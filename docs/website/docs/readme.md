---
title: README
description: dlt overview — extract, load, transform, and validate data with Python
keywords: [dlt, readme, overview, features, data loading, python]
---

# data load tool (dlt)

dlt is an open-source Python library that automates data loading. Drop it into a Google Colab notebook, AWS Lambda function, an Airflow DAG, your local laptop, or an LLM-assisted development workflow.

## Installation

dlt supports Python 3.9 through Python 3.14.

```sh
pip install dlt
```

## Quick start

Load chess game data from the chess.com API and save it in DuckDB:

```py
import dlt
from dlt.sources.helpers import requests

pipeline = dlt.pipeline(
    pipeline_name='chess_pipeline',
    destination='duckdb',
    dataset_name='player_data'
)

data = []
for player in ['magnuscarlsen', 'rpragchess']:
    response = requests.get(f'https://api.chess.com/pub/player/{player}')
    response.raise_for_status()
    data.append(response.json())

pipeline.run(data, table_name='player')
```

Try it out in the [Colab demo](https://colab.research.google.com/drive/1NfSB1DpwbbHX9_t5vlalBTf13utwpMGx?usp=sharing) or directly in the [playground](./tutorial/playground).

## Features

dlt provides lightweight Python interfaces to extract, load, inspect, and transform data. dlt and dlt docs are built from the ground up to be used with LLMs: the [LLM-native workflow](./dlt-ecosystem/llm-tooling/llm-native-workflow.md) will take your pipeline code to data in a notebook for over [5000 sources](https://dlthub.com/workspace).

### Extract from anything

- **[REST APIs](./tutorial/rest-api)** — declarative, strongly typed configuration with built-in pagination, auth, and [processing steps](./general-usage/resource.md#filter-transform-and-pivot-data) (`filter`, `map`) to reshape records before they reach the warehouse.
- **[SQL databases](./tutorial/sql-database)** — replicate tables with one line of config.
- **[Cloud storage](./tutorial/filesystem)** — load CSV, JSONL, and Parquet files from S3, GCS, Azure, R2, or local disk. Chain `filesystem()` with a reader and `pipeline.run()` in one pipe.
- **[DataFrames](./dlt-ecosystem/verified-sources/arrow-pandas.md)** — pass Pandas, Polars, or Arrow tables directly. dlt infers the schema from dtypes and moves Arrow-backed frames with zero copies.
- **[Python data structures](./tutorial/load-data-from-an-api)** — yield lists of dicts from a `@dlt.resource` and dlt infers the schema, types columns, and writes the table.
- **[And many more](./dlt-ecosystem/verified-sources)** verified sources.

### Load into 20+ destinations

dlt supports [20+ destinations](./dlt-ecosystem/destinations/) including DuckDB, Snowflake, BigQuery, Iceberg, and filesystem (S3, GCS, Azure, R2). Same resource, swap the destination string:

```py
dlt.pipeline(
    pipeline_name="luma",
    destination="duckdb",
#   destination="snowflake",
#   destination="iceberg",
#   destination="filesystem",   # S3, GCS, Azure, R2
).run(
    source=events(),
)
```

The [`@dlt.destination`](./dlt-ecosystem/destinations/destination) decorator lets you build custom sinks for reverse ETL pipelines.

### Schema inference, normalization, and evolution

- dlt infers [schemas](./general-usage/schema.md) and [data types](./general-usage/schema.md#data-types) from Python dicts, DataFrames, and Parquet files.
- [Nested JSON is normalized](./general-usage/schema.md#data-normalizer) into relational child tables with consistent naming.
- [Schema evolution](./general-usage/schema-evolution.md) handles `ALTER TABLE` on the fly, and [schema contracts](./general-usage/schema-contracts.md) let you freeze or discard unexpected columns.

### Automate pipeline maintenance

- **[Incremental loading](./general-usage/incremental-loading.md)** — track state with one decorator argument (`dlt.sources.incremental`).
- **Secrets and config** — `dlt.secrets.value` / `dlt.config.value` injection from TOML, env vars, or vaults across dev/staging/prod profiles.
- **Decorator knobs** — declare `write_disposition`, `primary_key`, `merge_key`, `file_format`, `schema_contract`, and more on `@dlt.resource`. Override any setting at runtime.

```py
@dlt.resource(
    name="guests",
    primary_key="api_id",
    write_disposition="merge",
    file_format="parquet",
    schema_contract={"columns": "freeze"},
)
def guests(
    updated_at=dlt.sources.incremental("updated_at"),
):
    yield from fetch_guests(since=updated_at.last_value)
```

### Read your data back — the Dataset API

Reconnect to any previously run pipeline with `dlt.attach` and access loaded tables as pandas, Arrow, or Ibis expressions:

```py
pipeline = dlt.attach(pipeline_name="my_pipeline")
table = pipeline.dataset().my_table

table.df()           # pandas DataFrame
table.arrow()        # pyarrow.Table — zero-copy into DuckDB / Polars
table.to_ibis()      # lazy Ibis expression, compiles to the destination's SQL dialect
```

### Transform with Ibis

Use [`@dlt.hub.transformation`](./dlt-ecosystem/transformations/) to write parameterized, composable transformations in Python that execute as SQL on the warehouse:

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

Transformations compose into sources — bundle them the same way you bundle ingest resources. Test locally on DuckDB, ship to Snowflake unchanged.

### Data quality checks

Validate loaded or transformed data with `dlthub.data_quality`. Checks compile to SQL and run where the data already lives — no extraction, no copies:

```py
import dlthub.data_quality as dq

results = dq.prepare_checks(
    pipeline.dataset().inventory,
    [
        dq.checks.is_not_null("price"),
        dq.checks.is_in("name", ["apple", "pear", "cherry"]),
    ],
    level="row",       # "row", "table", or "dataset"
).arrow()
```

Check results are Relations — load them back into the warehouse to trend pass rates, alert on regressions, and join verdicts against source rows.

### Inspect, deploy, and visualize

- dlt supports [Python and SQL data access](./general-usage/dataset-access/), [pipeline inspection](./general-usage/dashboard.md), and [visualizing data in Marimo Notebooks](./general-usage/dataset-access/marimo).
- dlt can be deployed anywhere Python runs, be it on [Airflow](./walkthroughs/deploy-a-pipeline/deploy-with-airflow-composer), [serverless functions](./walkthroughs/deploy-a-pipeline/deploy-with-google-cloud-functions), or any other cloud deployment of your choice.

## Examples

You can find examples for various use cases in the [code examples section](../examples/).

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

## License

`dlt` is released under the [Apache 2.0 License](https://github.com/dlt-hub/dlt/blob/devel/LICENSE.txt).
