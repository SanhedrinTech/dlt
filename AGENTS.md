# dlt principles
- This is a mature library used by thousands of the best engineers in the world
- dlt is a library, not a platform. users add it to their code. we respect existing workflows and other libraries they use
- no black boxes: clean, minimal Pythonic interfaces, human readable file formats, no side effects, documentation
- multiply - don't add: we do more work here so our users do less.

# dlt repo
- we use `uv` and `uv run`. look in @Makefile
- we branch from `devel` for feature branches

## Cursor Cloud specific instructions

### Overview
dlt is a Python ETL/ELT library (not a service). No external services are required for core development — DuckDB runs in-process as the default test destination.

### Running services
- **Lint**: `make lint` (runs mypy, ruff, flake8, bandit, docstring checks, lockfile check, and dependency checks). Subtargets: `make lint-core`, `make lint-security`, `make lint-docstrings`, `make lint-lock`, `make lint-deps`.
- **Format**: `make format` (black). Run `make format` before `make lint` to avoid formatting failures.
- **Tests (no external deps)**: `make test-common` — tests common, normalize, extract, pipeline, reflection, sources, workspace, libs, destinations modules. Use `make test-common-p` for parallel execution.
- **Tests (DuckDB + filesystem)**: `make test-load-local` — runs load tests against DuckDB and in-memory filesystem. No Docker needed.
- **Tests (specific file)**: `uv run pytest tests/path/to/test_file.py -v`
- **Pipeline smoke test**: `uv run pytest tests/pipeline/test_pipeline.py`
- See `Makefile` and `CONTRIBUTING.md` for the full list of test targets and CI details.

### Key caveats
- `make lint-core` reinstalls `docstring_parser_fork` (replaces `docstring_parser`). This is expected and intentional.
- The `lint-deps` target runs informational checks that may print warnings (prefixed with `-` in Makefile, so non-fatal).
- Tests that require Docker containers (postgres, clickhouse, etc.) need `make start-test-containers` first — see `CONTRIBUTING.md` "Local Destinations".
- `uv run` is preferred over activating the venv. All Makefile targets use `uv run` internally.
- Python 3.12 is installed on the Cloud VM. The project supports 3.9–3.14.