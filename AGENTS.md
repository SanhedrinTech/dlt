# dlt — data load tool

**dlt** is an open-source Python library for building scalable, maintainable data loading pipelines. It's used by thousands of the best engineers in the world to extract, normalize, and load data to various destinations.

## Core principles

- **Mature library, not a platform**: dlt is a library users add to their code. We respect existing workflows and other libraries they use.
- **No black boxes**: Clean, minimal Pythonic interfaces, human-readable file formats, no side effects, comprehensive documentation.
- **Multiply, don't add**: dlt does more work so users do less.
- **Pythonic and accessible**: Easy for beginners, powerful for advanced users. Full typing support.

## Project metadata

- **Version**: 1.27.0 (see `pyproject.toml`)
- **Python**: 3.9–3.13
- **License**: Apache 2.0
- **Package name**: `dlt` (PyPI)
- **Default branch**: `devel` (branch feature branches from here)
- **Package manager**: `uv` and `uv run` (see `@Makefile`)

## Repository structure

```
dlt/
├── dlt/                      # Main package
│   ├── common/               # Core utilities, schemas, configuration, storage
│   │   ├── configuration/    # Config specs, secrets, credentials
│   │   ├── data_types/       # Data type system and validation
│   │   ├── destination/      # Base interfaces for destinations
│   │   ├── incremental/      # Incremental load state management
│   │   ├── normalizers/      # Data normalization rules
│   │   ├── schema/           # Schema definition and evolution
│   │   ├── storages/         # Storage backends (local, s3, etc.)
│   │   ├── libs/             # Optional dependency wrappers (pandas, pyarrow, etc.)
│   │   └── ...               # utilities, typing, exceptions, logging, json
│   ├── extract/              # Data extraction (sources, resources, decorators)
│   │   ├── decorators.py     # @source, @resource, @transformer decorators
│   │   ├── source.py         # Source class and APIs
│   │   └── ...               # extraction logic
│   ├── normalize/            # Normalization pipeline, schema inference
│   │   └── items_normalizers/
│   ├── load/                 # Load phase orchestration
│   ├── pipeline/             # Pipeline state, orchestration, execution
│   │   ├── pipeline.py       # Main Pipeline class
│   │   ├── mark.py           # Marking API for advanced control
│   │   └── current.py        # Current pipeline context
│   ├── destinations/         # Destination implementations
│   │   ├── impl/             # 24+ destination implementations (snowflake, bigquery, postgres, etc.)
│   │   ├── dataset/          # Dataset/relation APIs for querying loaded data
│   │   └── decorators.py     # @destination decorator for custom destinations
│   ├── sources/              # Pre-built sources (sql_database, rest_api, etc.)
│   ├── dataset/              # Dataset and relation APIs
│   ├── helpers/              # Helper utilities and functions
│   ├── hub/                  # Integration with dltHub
│   ├── reflection/           # Type and signature reflection utilities
│   └── _workspace/           # Workspace/dashboard functionality
│
├── tests/                    # Comprehensive test suite
│   ├── common/               # Tests for common/ module
│   ├── extract/              # Tests for extraction
│   ├── normalize/            # Tests for normalization
│   ├── pipeline/             # Tests for pipeline functionality
│   ├── load/                 # Tests for load phase and destinations
│   │   ├── <destination>/    # Destination-specific load tests
│   │   └── sources/          # Source integration tests
│   ├── sources/              # Tests for pre-built sources
│   ├── workspace/            # Workspace/dashboard tests
│   ├── e2e/                  # End-to-end tests
│   ├── helpers/              # Helper tests
│   └── .dlt/                 # Test pipeline states and artifacts
│
├── docs/                     # Documentation (Markdown, website, examples)
├── deploy/                   # Docker and deployment configurations
├── tools/                    # Internal tooling scripts
├── .claude/                  # Claude Code configuration
│   └── rules/                # AI-facing style and convention rules
└── .github/                  # GitHub workflows and templates
```

## Core modules and responsibilities

### `dlt.common` — Shared utilities and infrastructure
Core building blocks used throughout dlt. No heavy dependencies (except optional libs).

**Key submodules:**
- **configuration**: Configuration specs, secrets management, environment variable loading
- **schema**: Schema definition, normalization rules, schema evolution, type mapping
- **data_types**: dlt's type system, data type decorators, SQL type mapping
- **normalizers**: JSON normalization rules, flattening, schema inference
- **destination**: Base interfaces (`DestinationCapabilities`, `LoadJob`)
- **storages**: File storage abstraction (filesystem, s3, gs, az, etc.)
- **libs**: Wrappers for optional dependencies (pandas, pyarrow, sqlalchemy, etc.)

### `dlt.extract` — Source and resource definitions
Decorators and APIs for defining data sources and resources.

**Public APIs:**
- `@source` — Decorator for grouping resources, managing authentication
- `@resource` — Decorator for individual data streams
- `@transformer` — Decorator for transforming data from another resource
- `@defer` — Decorator for lazy evaluation
- `Source` class — Programmatic source definition

### `dlt.normalize` — Schema inference and normalization
Converts raw data into normalized, schematized form. Runs locally or on destination.

### `dlt.load` — Load phase execution
Orchestrates the load phase: staging, type mapping, destination-specific load operations.

### `dlt.pipeline` — Pipeline orchestration
Central API for orchestrating E→N→L workflows. Manages state, retries, incremental loading.

**Key classes:**
- `Pipeline` — Main orchestration class
- `progress` — Progress tracking callbacks
- `mark` — Advanced marking API
- `current` — Context manager for accessing active pipeline

### `dlt.destinations` — Destination implementations and APIs
24+ destination implementations and shared destination infrastructure.

**Structure:**
- `impl/` — Each destination (snowflake, bigquery, postgres, duckdb, etc.)
- `dataset/` — Query APIs after loading (`Dataset`, `Relation`)
- `decorators.py` — `@destination` decorator for custom destinations

**Supported destinations:** Snowflake, BigQuery, Redshift, Postgres, DuckDB, Databricks, ClickHouse, Delta Lake, Polars, Parquet, S3, GCS, Azure, Weaviate, Qdrant, Lancedb, and more.

### `dlt.sources` — Pre-built source connectors
Ready-to-use sources for common platforms.

**Key sources:**
- `sql_database` — SQL database connector (SQLAlchemy-based)
- `rest_api` — REST API paginated requests
- `filesystem` — File system data
- Cloud integrations (Slack, Google Analytics, Zendesk, etc. — see docs)

### `dlt.hub` — dltHub integration
APIs for managing pipelines on dltHub platform.

### `dlt.dataset` — Query APIs
Query data after it's been loaded (currently duckdb-based).

## Development setup

### Prerequisites
- Python 3.9–3.13
- `uv` package manager (install: `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Git

### Installation

```bash
# Clone the repository
git clone https://github.com/dlt-hub/dlt.git
cd dlt

# Install development dependencies
make dev

# Or with specific extras (airflow, hub, etc.)
make dev-airflow
make dev-hub
```

The `make dev` target uses `uv sync --all-extras` to install all optional dependencies and development groups.

### Verify installation

```bash
uv run python -c "import dlt; print(dlt.__version__)"
uv run pytest tests/common -v
```

## Git workflow

### Branching
- **Base branch**: `devel`
- **Feature branches**: Branch from `devel` with descriptive names (e.g., `feature/incremental-load`, `fix/schema-validation`)
- **Release branches**: Tagged commits on `devel` (semver)

### Commit conventions
- Clear, descriptive commit messages
- Reference issues when applicable (e.g., "fix: handle null values in schema inference (closes #1234)")
- Keep commits focused and atomic

### Pull requests
- Target `devel`
- Include a description of changes and motivation
- Link related issues
- Ensure tests pass before merging

## Code conventions

### Import order
Groups separated by blank lines:
1. **stdlib**: `import os`, `from typing import ...`
2. **third-party**: `import pendulum`, `from sqlalchemy import ...`
3. **local dlt**: `from dlt.common import ...`, `from dlt.extract import ...`
4. (tests only) **test utilities**: `from tests.utils import ...`

Rules:
- Stdlib imports are always module-level
- `dlt.common` imports (except `dlt.common.libs`) are module-level
- `dlt.common.libs.*` imports may be inline (wrap optional dependencies)
- No relative imports

**Optional dependencies**: Always import from `dlt.common.libs.<package>`, never directly:
```python
from dlt.common.libs.pandas import pandas
from dlt.common.libs.pyarrow import pyarrow as pa
```

See `@.claude/rules/imports.md` for full details.

### Logging
```python
from dlt.common import logger
logger.info("message")  # never: import logging
```

### Code style
- **Line length**: 100 characters (enforced by Black and isort)
- **Formatter**: `black` and `ruff`
- **Type checker**: `mypy`
- **Linter**: `ruff` + `flake8` (excluding D docstring checks)
- **Security**: `bandit`

### Types and type annotations
- Use full generic parametrization: `Dict[str, Any]`, not `dict`
- Type aliases use `T`-prefix PascalCase: `TDataItem`, `TLoaderFileFormat`
- Use `TypedDict` for structured dicts, `NamedTuple` for lightweight immutable data
- Prefer `functools.partial` over `lambda`
- All public APIs must be typed

### Exceptions
- Inherit from `DltException` (or mixin `TerminalException`/`TransientException`)
- See `@dlt/common/exceptions.py` for patterns

### Comments and docstrings
- **Comments**: Inline comments use lowercase and explain WHY, not WHAT
- **Docstrings**: Google style, three levels of detail:
  - Internal methods: short form (description only)
  - Public API: full form (Args, Returns, Raises, Yields)
  - Never state the obvious or document thought process

See `@.claude/rules/docstrings.md` for full format and examples.

## Testing

### Test organization
Tests mirror the source structure:
- `tests/common/` — Common module tests
- `tests/extract/` — Extraction tests
- `tests/normalize/` — Normalization tests
- `tests/pipeline/` — Pipeline tests
- `tests/load/` — Load and destination tests
- `tests/sources/` — Pre-built source tests
- `tests/workspace/` — Workspace and dashboard tests
- `tests/e2e/` — End-to-end integration tests

### Running tests locally

```bash
# Run all common tests
make test-common

# Run in parallel (recommended)
make test-common-p

# Run specific test file
uv run pytest tests/common/test_schema.py -v

# Run with keyword filter
uv run pytest tests/pipeline -k "incremental" -v

# Run with specific markers
uv run pytest tests/ -m "not forked and not serial" -x
```

### Test markers
- `serial` — Must run serially (state isolation)
- `forked` — Requires process forking
- `essential` — Essential tests for remote destinations

### CI test matrix (see `@Makefile`)
The CI pipeline runs granular test suites based on installed extras:
- `test-common-core` — Minimal core tests
- `test-pipeline-min` — Pipeline smoke tests with duckdb
- `test-load-local` — Load tests with duckdb + filesystem
- `test-dest-remote-essential` — Remote destination essentials
- Full matrix includes destination-specific, source-specific, and feature-specific suites

### Writing tests
- Use `pytest` fixtures for setup/teardown
- Mark expensive tests (remote, integration) with `@pytest.mark.essential`
- Test both happy path and error cases
- Avoid mocks for internal code; use real instances
- Use `tests/utils.py` for common test utilities

## Linting and formatting

```bash
# Format code
make format

# Run all linters
make lint

# Run specific linters
make lint-core          # mypy, ruff, flake8
make lint-security      # bandit
make lint-docstrings    # docstring checks on public API
make lint-deps          # dependency checks
make lint-lock          # uv lockfile validation
```

These are run in CI before merge. Fix any failures before pushing.

## Building and publishing

```bash
# Build distribution packages
make build-library

# Publish to PyPI (requires token)
make publish-library

# Test build (creates Docker images)
make test-build-images
```

## Key public APIs

All exports in `@dlt/__init__.py` are part of the public API. Changes here are breaking changes.

**Core APIs:**
```python
import dlt

# Decorators
@dlt.source
@dlt.resource
@dlt.transformer
@dlt.defer
@dlt.destination

# Pipeline
pipeline = dlt.pipeline("my_pipeline", destination="duckdb")
pipeline.run(my_source())

# Configuration
dlt.config           # Access configuration
dlt.secrets          # Access secrets

# Schema and types
dlt.Schema
dlt.TSecretValue
dlt.TCredentials

# Advanced
dlt.mark             # Marking API
dlt.current          # Current pipeline context
dlt.dbt              # dbt integration
dlt.hub              # dltHub APIs
```

## Backward compatibility

- **Breaking changes** must be flagged with `# BREAKING:` comments
- Use deprecation warnings (`DltDeprecationWarning`) for soft deprecations
- Test backward compatibility with `@tests/pipeline/test_dlt_versions.py`
- Schema/state migrations use `engine_version` and versioned layouts

See `@.claude/rules/public-api.md` for full policy.

## AI assistant conventions

### When working on dlt

1. **Read the rules first**: Check `@.claude/rules/` for conventions on imports, docstrings, coding style, and public API.
2. **Mirror existing patterns**: Look at similar code in the same directory before writing new code.
3. **Test thoroughly**: Run `make test-common-p` and relevant destination tests before pushing.
4. **Type everything**: All parameters and return types must be annotated for public APIs.
5. **Lint before push**: Run `make lint-core` and fix any issues.
6. **Branch from `devel`**: Always start from the latest `devel`, not from `main` or other branches.
7. **Document public APIs**: Full Google-style docstrings with Args, Returns, Raises for public functions/classes.
8. **No breaking changes without discussion**: Changes to anything in `@dlt/__init__.py` are breaking changes.
9. **Consider downstream users**: dlt is used by thousands of engineers; side effects and surprises are costly.

### Directory conventions
- `dlt/common/` — Shared, lightweight utilities (always importable)
- `dlt/extract/` — Source/resource definitions and decorators
- `dlt/normalize/` — Schema inference and normalization logic
- `dlt/load/` — Load phase, staging, type mapping
- `dlt/pipeline/` — Orchestration and state management
- `dlt/destinations/` — Destination implementations
- `dlt/sources/` — Pre-built source connectors
- `dlt/_workspace/` — Workspace/dashboard features
- `dlt/helpers/` — Non-critical helper functions

### Common tasks

**Adding a new destination:**
1. Create `dlt/destinations/impl/<dest_name>/`
2. Implement `Destination` interface from `dlt/common/destination/`
3. Add load job implementations
4. Add type mapping for destination-specific types
5. Create `tests/load/<dest_name>/` test suite
6. Document in `docs/`

**Adding a new pre-built source:**
1. Create `dlt/sources/<source_name>/`
2. Use `@source` and `@resource` decorators
3. Add tests in `tests/sources/<source_name>/`
4. Example: `dlt/sources/sql_database/`

**Modifying schema handling:**
- Schema updates go in `dlt/common/schema/`
- Ensure backward compatibility with `engine_version`
- Add tests in `tests/common/test_schema.py`

**Adding optional dependency support:**
1. Add to `pyproject.toml` optional-dependencies
2. Create wrapper in `dlt/common/libs/<package>.py`
3. Raise `MissingDependencyException` if not installed
4. Import from wrapper in code, not directly
5. Example: `dlt/common/libs/pandas.py`

## Configuration and environment

dlt uses:
- **Configuration**: YAML/TOML files in `.dlt/` or environment variables (with `DLTDLT_` prefix for internal config, or provider-specific prefixes)
- **Secrets**: `.dlt/secrets.toml` or environment variables (with `DLTDLT_` prefix)
- **State**: Local JSON files in `.dlt/.state/` by default, customizable per destination

Example:
```toml
# .dlt/secrets.toml
[sources.api]
api_key = "..."

[destination]
connection_string = "postgresql://..."
```

See `@dlt/common/configuration/` for implementation.

## Performance considerations

- dlt is designed to be scalable; large datasets (billions of rows) should load efficiently
- Normalization can run on the destination (faster for large data)
- Incremental loading reduces data transfer and compute
- Staging is used for safety and efficiency; prefer destination-native staging
- Avoid memory-intensive operations on large datasets in Python; push to destination when possible

## Common patterns and examples

### Simple pipeline
```python
import dlt
from dlt.sources.helpers import requests

@dlt.resource
def get_users():
    yield requests.get("https://api.example.com/users").json()

pipeline = dlt.pipeline("my_pipeline", destination="duckdb")
pipeline.run(dlt.source(get_users))
```

### Incremental loading
```python
@dlt.resource(primary_key="id")
def get_orders(
    created_at: dlt.sources.config.value = dlt.config.value
):
    dlt_state = dlt.current.source_state()
    since = dlt_state.get("since", created_at)
    
    data = requests.get(
        "https://api.example.com/orders",
        params={"since": since}
    ).json()
    
    dlt_state["since"] = max(item["created_at"] for item in data)
    yield data
```

### Custom destination
```python
@dlt.destination(name="my_dest")
def write_to_custom(items, table):
    for item in items:
        # custom logic
        pass
```

## Useful make targets

```bash
make help              # Show all make targets
make dev               # Install all extras + dev dependencies
make lint              # Run all linters
make format            # Format code with black
make test-common-p     # Run common tests in parallel
make build-library     # Build distribution packages
make start-test-containers  # Start postgres, clickhouse, etc. for local testing
make update-cli-docs   # Regenerate CLI docs
```

## References and documentation

- **Main docs**: https://dlthub.com/docs
- **GitHub**: https://github.com/dlt-hub/dlt
- **PyPI**: https://pypi.org/project/dlt
- **Issues**: https://github.com/dlt-hub/dlt/issues
- **Code style rules**: `@.claude/rules/`
- **Makefile**: `@Makefile` (all build, test, lint, publish targets)
- **Dependencies**: `@pyproject.toml`
- **Version**: `@dlt/version.py`

---

**Last updated**: 2025-07-14  
**Version**: dlt 1.27.0  
**Maintainers**: Marcin Rudolf, Adrian Brudaru, Anton Burnashev, David Scharf