---
title: Validate data with Pydantic models
description: Enforce data types, reject bad records, and catch schema violations during extraction
keywords: [pydantic, validation, schema contracts, data quality, how-to]
---

# Validate data with Pydantic models

This guide shows you how to attach a Pydantic model to a dlt resource so that every
record is validated at extraction time. You will define expected types, choose what
happens when a record breaks the rules, and handle validation errors in your pipeline
code.

Make sure you have [installed dlt](../reference/installation.md) before
following the steps below. You also need Pydantic v2 or higher (`pip install pydantic`).

## 1. Define a Pydantic model for your data

Create a Pydantic model that describes the shape of one record. Each field maps to a
destination column.

```py
from typing import Optional
from pydantic import BaseModel


class User(BaseModel):
    id: int
    name: str
    email: Optional[str] = None
    is_active: bool = True
```

A few rules dlt follows when reading the model:

- `Optional` fields become **nullable** columns.
- `list`, `dict`, and nested `BaseModel` fields are stored as JSON by default. To create
  nested tables instead, see [step 5](#5-control-nested-types).
- Type mappings follow the
  [standard dlt type system](../general-usage/schema.md).

## 2. Attach the model to a resource

Pass the model to the `columns` parameter of `@dlt.resource`. dlt adds a validation
step that converts every yielded item into a model instance before loading.

```py
import dlt
from typing import Iterator, Optional
from pydantic import BaseModel


class User(BaseModel):
    id: int
    name: str
    email: Optional[str] = None
    is_active: bool = True


@dlt.resource(name="users", columns=User)
def users_resource() -> Iterator[dict]:
    yield {"id": 1, "name": "Alice", "email": "alice@example.com", "is_active": True}
    yield {"id": 2, "name": "Bob", "email": None, "is_active": False}


pipeline = dlt.pipeline(
    pipeline_name="validated_users",
    destination="duckdb",
)

info = pipeline.run(users_resource())
print(info)
```

Run this pipeline and both records load successfully because they match the `User`
model.

## 3. Choose how schema violations are handled

When dlt sees a Pydantic model on a resource it sets an **implicit schema contract**:

| Entity | Default mode | Effect |
|--------|-------------|--------|
| `tables` | `evolve` | New tables are created as needed |
| `columns` | `discard_value` | Extra fields not in the model are silently dropped |
| `data_type` | `freeze` | A record with a wrong type raises an error |

You can override any of these modes with the `schema_contract` parameter:

```py
@dlt.resource(
    name="users",
    columns=User,
    schema_contract={"columns": "freeze", "data_type": "discard_row"},
)
def users_resource() -> Iterator[dict]:
    ...
```

The four available modes are:

| Mode | Behavior |
|------|----------|
| `evolve` | Accept the change and update the schema |
| `freeze` | Raise a `DataValidationError` |
| `discard_row` | Drop the entire record silently |
| `discard_value` | Drop the offending field but keep the record |

:::note
`data_type: discard_value` is **not supported** when using Pydantic models. Use
`discard_row` or `freeze` for the `data_type` entity.
:::

For the full contract reference, see
[schema and data contracts](../general-usage/schema-contracts.md).

## 4. Handle validation errors

When `data_type` is set to `freeze` (the default with Pydantic), a bad record raises
`DataValidationError` wrapped inside `PipelineStepFailed`. Catch it to inspect what
went wrong:

```py
import dlt
from typing import Iterator, Optional
from dlt.pipeline.exceptions import PipelineStepFailed
from pydantic import BaseModel


class User(BaseModel):
    id: int
    name: str
    email: Optional[str] = None
    is_active: bool = True


@dlt.resource(name="users", columns=User)
def users_resource() -> Iterator[dict]:
    yield {"id": 1, "name": "Alice", "email": "alice@example.com", "is_active": True}
    # This record has a wrong type for "id"
    yield {"id": "not_a_number", "name": "Bad Record", "email": None, "is_active": True}


pipeline = dlt.pipeline(
    pipeline_name="validated_users",
    destination="duckdb",
)

try:
    pipeline.run(users_resource())
except PipelineStepFailed as e:
    if "DataValidationError" in str(type(e.__cause__).__name__):
        print(f"Validation failed: {e.__cause__}")
    else:
        raise
```

If you prefer to **skip bad records** instead of stopping, set `data_type` to
`discard_row`:

```py
@dlt.resource(
    name="users",
    columns=User,
    schema_contract={"data_type": "discard_row"},
)
def users_resource() -> Iterator[dict]:
    ...
```

With `discard_row`, records that fail Pydantic validation are silently dropped and
the pipeline continues.

## 5. Control nested types

By default, `list`, `dict`, and nested `BaseModel` fields are stored as a single JSON
column. If you want dlt to create separate nested tables instead, set
`skip_nested_types` in a `DltConfig` class variable on your model:

```py
from typing import ClassVar, List, Optional
from pydantic import BaseModel

from dlt.common.libs.pydantic import DltConfig


class Address(BaseModel):
    street: str
    city: str


class User(BaseModel):
    dlt_config: ClassVar[DltConfig] = {"skip_nested_types": True}

    id: int
    name: str
    email: Optional[str] = None
    addresses: List[Address] = []
```

With `skip_nested_types` set to `True`, dlt omits `addresses` from the Pydantic schema
and falls back to its default behavior of creating a child table `users__addresses`.

## 6. Use custom validators

Pydantic `field_validator` and `model_validator` decorators work with dlt. Use them
to enforce business rules beyond type checking:

```py
from typing import Optional
from pydantic import BaseModel, field_validator


class User(BaseModel):
    id: int
    name: str
    email: Optional[str] = None
    age: int

    @field_validator("age")
    @classmethod
    def age_must_be_positive(cls, v: int) -> int:
        if v < 0:
            raise ValueError("age must be positive")
        return v
```

When a record fails a custom validator, the behavior follows your `data_type` contract
mode: `freeze` raises an error, `discard_row` drops the record.

## 7. Next steps

- Learn about all contract modes and how they interact:
  [schema and data contracts](../general-usage/schema-contracts.md)
- Use discriminated unions to validate heterogeneous event streams:
  [discriminated unions](../general-usage/schema-contracts.md#validating-event-streams-with-pydantic)
- Explore the full data quality lifecycle:
  [data quality lifecycle](../general-usage/data-quality-lifecycle.md)
- Define schemas without Pydantic using column hints:
  [resource](../general-usage/resource.md#define-a-schema-with-pydantic)
