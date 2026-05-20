---
title: README
description: dlt overview — extract, load, transform, and validate data with Python
keywords: [dlt, readme, overview, features, data loading, python, etl, data pipeline]
---

# data load tool (dlt)

dlt is an open-source Python library that automates data loading. Drop it into a Google Colab notebook, AWS Lambda function, an Airflow DAG, your local laptop, or an LLM-assisted development workflow.

## Installation

dlt supports Python 3.9 through Python 3.14.

```sh
pip install dlt
```

Try it out in the [Colab demo](https://colab.research.google.com/drive/1NfSB1DpwbbHX9_t5vlalBTf13utwpMGx?usp=sharing) or directly in the [playground](./tutorial/playground).

## Extract from anything

dlt extracts data from [REST APIs](./tutorial/rest-api), [SQL databases](./tutorial/sql-database), [cloud storage](./tutorial/filesystem), [DataFrames](./dlt-ecosystem/verified-sources/arrow-pandas.md), [Python data structures](./tutorial/load-data-from-an-api), and [many more](./dlt-ecosystem/verified-sources) verified sources. The examples below walk through a real-world pipeline that loads event data from the Luma API, then builds on it step by step.

### REST APIs — declarative and strongly typed

Define the API endpoints, pagination, and auth. dlt handles the rest:

```py
from dlt.sources.rest_api import rest_api_source

luma = rest_api_source({
    "client": {
        "base_url": "https://api.lu.ma/public/v1",
        "paginator": {"type": "cursor", "cursor_path": "next_cursor"},
    },
    "resources": [
        {
            "name": "guests",
            "endpoint": {
                "path": "event/get-guests",
            },
        },
        {
            "name": "events",
            "endpoint": {
                "path": "event/get",
            },
        },
    ],
})
```

Typed, declarative primitives combined with [dlt context](https://dlthub.com/workspace) enable one-shot pipelines with LLMs.

### Filter, map, and flatten at the source

Not every record from an API is useful and not every record shape matches the table you want. Chain `filter` and `map` processing steps to drop noise and reshape records before they reach the warehouse:

```py
from dlt.sources.rest_api import rest_api_source

luma = rest_api_source({
    "client": {
        "base_url": "https://api.lu.ma/public/v1",
    },
    "resources": [
        {
            "name": "guests",
            "endpoint": {
                "path": "event/get-guests",
            },
            "processing_steps": [
                {"filter": lambda r: r["approval_status"] == "approved"},
                {"map": flatten_guest},
            ],
        },
    ],
})
```

The `flatten_guest` function is plain Python — unnest the `guest` object, normalize the email, derive new columns, and parse timestamps. Declarative config for the pipeline shape, imperative Python when you need to transform:

```py
from typing import Any
from datetime import datetime

def flatten_guest(record: dict[str, Any]) -> dict[str, Any]:
    guest = record.pop("guest", {})
    email = (guest.get("email") or "").lower()
    return {
        **record,
        **guest,
        "email": email,
        "email_domain": email.rsplit("@", 1)[-1] if "@" in email else None,
        "registered_at": datetime.fromisoformat(record["registered_at"]),
        "is_checked_in": bool(record.get("checked_in_at")),
    }
```

### `@dlt.resource` — the smallest building block

A resource is any iterable of records. Yield a list of dicts and dlt infers the schema, types the columns, and writes the table. When the declarative REST API source is not enough, drop into plain Python with the same building blocks:

```py
import dlt

@dlt.resource
def events():
    yield [
        {
            "one": {
                "two": {"three": "value"}
            }
        }
    ]
```

Build a `RESTClient` once, share it across every resource, and each body collapses to a single `yield from`.

### `@dlt.source` — grouping resources into an integration

A source is a function that returns one or more resources. Think of it as the integration (Luma, Stripe, GitHub) while each resource is a single stream of records (events, guests). One source produces one schema with many tables.

Use closures to share a client, auth, and base URL across resources. Secrets resolve at runtime with `dlt.secrets.value` from TOML, env vars, or vaults — the same code runs across dev, staging, and prod:

```py
@dlt.source
def luma_source(api_key=dlt.secrets.value):
    client = RESTClient(
        base_url="https://api.lu.ma/public/v1",
    )

    @dlt.resource(primary_key="api_id")
    def events():
        yield from client.paginate("events")

    @dlt.resource
    def guests():
        yield from client.paginate("guests")

    return [events, guests]
```

```py
dlt.pipeline(
    pipeline_name="luma",
    destination="duckdb",
    dataset_name="luma_data",
).run(luma_source())
```

### Decorator knobs

Declare identity, load behavior, schema, and incremental loading directly on the decorator. Decorators let you express intent — no need to hand-code merge strategies, primary key lookups, or schema contracts. Every setting can be overridden at runtime via `pipeline.run()` or `config.toml`:

```py
@dlt.resource(
    # ── IDENTITY ─────────────────────────────────
    name="guests",
    primary_key="api_id",
    merge_key=("event_id", "api_id"),
    table_name=lambda row: f"guests_{row['type']}",   # dynamic routing

    # ── LOAD BEHAVIOR ────────────────────────────
    write_disposition="merge",
    file_format="parquet",
    parallelized=True,

    # ── SCHEMA ───────────────────────────────────
    columns={"email": {"x-annotation-pii": True}},
    schema_contract={"columns": "freeze"},
)
def guests(
    # ── INCREMENTAL ──────────────────────────────
    updated_at=dlt.sources.incremental("updated_at"),
):
    yield from fetch_guests(since=updated_at.last_value)
```

### DataFrames — Pandas, Polars, Arrow

Load DataFrames directly. dlt infers the schema from dtypes, preserves timestamps, decimals, and nested types end-to-end, and moves Arrow-backed frames with zero copies. Append, replace, or merge the same way you would any other resource:

```py
import dlt
import pandas as pd

df = pd.DataFrame({
    "event":   ["dlt summit 2026", "DuckCon", "Iceberg Day"],
    "signups": [1240, 860, 410],
})

dlt.pipeline(
    pipeline_name="events",
    destination="duckdb",
    dataset_name="event_data",
).run(
    df,
    table_name="top_events",
)
```

### Cloud storage and files — CSV, JSONL, Parquet

A three-step flow — list the files, parse them, load a table. `filesystem()` enumerates files from local disk, S3, GCS, or Azure. Pipe them into a reader and `pipeline.run()` writes the table with schema inferred:

```py
from dlt.sources.filesystem import filesystem, read_csv_duckdb

files = filesystem(
    bucket_url="file://data",
    file_glob="*.csv",
)

source = (
    files | read_csv_duckdb()
).with_name("guests")

dlt.pipeline(
    pipeline_name="files",
    destination="duckdb",
    dataset_name="file_data",
).run(source)
```

## Load into 20+ destinations

Same source, swap the destination string. dlt handles credentials, `CREATE TABLE` in the target dialect, type conversions, staging to S3/GCS for warehouses that need it, and schema drift with `ALTER TABLE` on the fly:

```py
dlt.pipeline(
    pipeline_name="luma",
    destination="duckdb",
#   destination="snowflake",
#   destination="iceberg",
#   destination="filesystem",   # S3, GCS, Azure, R2
).run(
    source=luma_source(),
)
```

Supported destinations include DuckDB, Snowflake, BigQuery, Iceberg, Databricks, Postgres, Redshift, and [many more](./dlt-ecosystem/destinations/).

The [`@dlt.destination`](./dlt-ecosystem/destinations/destination) decorator lets you build custom sinks for reverse ETL pipelines.

## Read your data back

### `dlt.attach` — reconnect to a pipeline

A pipeline is durable — `dlt.attach` reconnects to one by name. You get back the same schema, destination, and dataset you had at load time. No re-running, no re-ingesting:

```py
import dlt

pipeline = dlt.attach(
    pipeline_name="luma",
    destination="duckdb",
    dataset_name="luma_data",
)

pipeline.destination.destination_type   # "duckdb", "snowflake", ...
pipeline.dataset().tables               # tables in the loaded dataset
```

### The Dataset API

Every loaded table is reachable as `pipeline.dataset().<table>`. Pick the format that matches your tool — Arrow for zero-copy into DuckDB or Polars, pandas for notebooks, Ibis for lazy expressions that compile to SQL on the warehouse:

```py
guests = pipeline.dataset().guests

guests.arrow()        # pyarrow.Table
guests.df()           # pandas DataFrame
guests.to_ibis()      # Ibis expression — lazy, composable
```

The schema is introspectable — render relations straight from the loaded schema with no extra modeling layer.

## Data quality — checks as data

### Define checks

Checks compile to SQL and run where the data already lives — no extraction, no copies. `prepare_checks` returns a `dlt.Relation`. Levels control the verdict's grain: `row` for one verdict per record, `table` for one per table, `dataset` for cross-table assertions:

```py
import dlthub.data_quality as dq

inventory_checks = [
    dq.checks.is_in("name", ["apple", "pear", "cherry"]),
    dq.checks.is_not_null("price"),
    dq.checks.case("price < 0"),     # arbitrary SQL predicate, row-wise
]

results = dq.prepare_checks(
    pipeline.dataset().inventory,
    inventory_checks,
    level="row",       # "row", "table", or "dataset"
).arrow()
```

### `CheckSuite` — inspect successes and failures

A `CheckSuite` bundles checks per table and runs them on the dataset. Inspect the rows behind each verdict — every method returns a Relation:

```py
check_suite = dq.CheckSuite(
    pipeline.dataset(),
    checks={"inventory": inventory_checks},
)

check_suite.get_successes("inventory", "name__is_in").arrow()
check_suite.get_failures("inventory",  "name__is_in").arrow()
```

### Persist check results

Check results are Relations, so they load like any other resource. Store them in the same warehouse or ship them elsewhere — then trend pass rates, alert on regressions, and join quality results back against the rows they describe:

```py
pipeline.run(
    [
        dq.prepare_checks(
            pipeline.dataset().inventory,
            inventory_checks,
            level="row",
        ).arrow()
    ],
    table_name="dlt_data_quality",
)
```

## Transformations

### Ibis — Python in, SQL out

`.to_ibis()` lifts a loaded table into an Ibis expression. Compose group-bys, joins, and window functions in Python — nothing runs until you materialize with `.to_pyarrow()`, `.to_pandas()`, or `.to_polars()`. The compiler emits SQL in the destination's dialect and pushes the work down:

```py
customers = pipeline.dataset().customers.to_ibis()

customer_cities = (
    customers
    .group_by("city")
    .aggregate(number_of_customers=ibis._.id.count())
)

customer_cities                # the query plan (lazy)
customer_cities.to_pyarrow()   # execute on the warehouse
```

### `@dlt.hub.transformation` — parameterized transforms

Same decorator pattern — input is a `dlt.Dataset`, output yields Ibis tables that dlt materializes as resources. Because the transform is parameterized, you can run the same logic across dev, staging, and prod datasets, chain transformations where each one's output feeds the next, and test locally on DuckDB then ship to Snowflake unchanged:

```py
@dlt.hub.transformation
def customer_payments(dataset: dlt.Dataset):
    orders   = dataset.table("orders").to_ibis()
    payments = dataset.table("payments").to_ibis()

    yield (
        payments
        .left_join(orders, payments.order_id == orders.id)
        .group_by(orders.customer_id)
        .aggregate(total_amount=ibis._.amount.sum())
    )
```

### Compose transformations as a source

A source bundles transformations the same way it bundles ingest resources. Raw data lands in DuckDB or Iceberg (wherever is cheap to land) and modeled marts get their own pipeline and destination. Same primitives at every layer — `@dlt.resource`, `@dlt.source`, `@dlt.hub.transformation`:

```py
@dlt.source
def customers_metrics(raw_dataset: dlt.Dataset):
    return [
        customer_payments(raw_dataset),
        another_transformation(raw_dataset),
    ]

new_pipeline = dlt.pipeline(
    "customer_metrics",
    destination="snowflake",
)

new_pipeline.run(
    customers_metrics(original_pipeline.dataset())
)
```

### Close the loop — quality on modeled data

The data quality API works on any dataset — whether the rows came from a REST API, a CSV, or an Ibis aggregation. Ingest, transform, validate: one mental model, one toolkit:

```py
dq.prepare_checks(
    customers_metrics(original_pipeline.dataset()),
    [
        dq.checks.case("number_of_orders > 20"),
        dq.checks.is_not_null("customer_id"),
    ],
    level="dataset",
).df()
```

## Deploy anywhere

dlt is a library, not a platform. There is no proprietary runtime to lock into and no managed service required — it runs anywhere Python runs. Add it to the orchestrator you already use:

- [Airflow](./walkthroughs/deploy-a-pipeline/deploy-with-airflow-composer)
- [GitHub Actions](./walkthroughs/deploy-a-pipeline/deploy-with-github-actions)
- [Google Cloud Functions](./walkthroughs/deploy-a-pipeline/deploy-with-google-cloud-functions)
- [Dagster](./walkthroughs/deploy-a-pipeline/deploy-with-dagster)
- [Modal](./walkthroughs/deploy-a-pipeline/deploy-with-modal)
- [Kestra](./walkthroughs/deploy-a-pipeline/deploy-with-kestra)
- [Prefect](./walkthroughs/deploy-a-pipeline/deploy-with-prefect)

See the full list in the [deploy guide](./walkthroughs/deploy-a-pipeline/).

## dltHub Pro

For teams that want managed scheduling, observability, and deployment on top of dlt's open-source building blocks, [dltHub Pro](https://dlthub.com/docs/hub/introduction) provides a production runtime with agent-facing toolkits. It pairs the pipelines you build locally with managed infrastructure — secrets management, OTEL telemetry, transform-aware triggers, and a deployment agent — so the path from laptop to production is one command:

```sh
uvx dlthub-start
```

Pro also unlocks source-available features including dltHub transformations, the Iceberg destination, and MS SQL Change Tracking. Over 10,000 companies run dlt in production today; Pro is built for the smallest team that can run an end-to-end data stack.

See the [dltHub Pro docs](./hub/introduction) or sign up at [app.dlthub.com](https://app.dlthub.com).

## Examples

Find examples for various use cases in the [code examples section](../examples/).

## Adding as a dependency

`dlt` follows semantic versioning with the `MAJOR.MINOR.PATCH` pattern.

- `major` — breaking changes and removed deprecations
- `minor` — new features, sometimes automatic migrations
- `patch` — bug fixes

Allow only `patch` level updates automatically using the compatible release specifier. For example `dlt~=1.23.0` allows versions `>=1.23.0` and less than `<1.24.0`.

## Get involved

- **Connect with the community**: Join other dlt users and contributors on [Slack](https://dlthub.com/community).
- **Report issues and suggest features**: Use [GitHub Issues](https://github.com/dlt-hub/dlt/issues) to report bugs or suggest new features. Search the tracker for possible duplicates first.
- **Track progress**: Check out the [public GitHub project](https://github.com/orgs/dlt-hub/projects/9).
- **Contribute code**: Read the [contributing guide](https://github.com/dlt-hub/dlt/blob/devel/CONTRIBUTING.md) before opening a PR.

## Sponsors

[Blacksmith](https://blacksmith.sh/?utm_source=dlt&utm_medium=readme&utm_campaign=sponsorship) is a CI/CD platform that accelerates GitHub Actions by reducing queue times and improving build performance. It helps teams run workflows faster and more efficiently, making it easier to maintain a smooth development pipeline. dltHub is grateful to Blacksmith for sponsoring dlt with free CI/CD minutes, which helps keep builds fast and costs lower.

## License

`dlt` is released under the [Apache 2.0 License](https://github.com/dlt-hub/dlt/blob/devel/LICENSE.txt).
