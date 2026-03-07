# dlt principles
- This is a mature library used by thousands of the best engineers in the world
- dlt is a library, not a platform. users add it to their code. we respect existing workflows and other libraries they use
- no black boxes: clean, minimal Pythonic interfaces, human readable file formats, no side effects, documentation
- multiply - don't add: we do more work here so our users do less.

# dlt repo
- we use `uv` and `uv run`. look in @Makefile
- we branch from `devel` for feature branches

## Cursor Cloud specific instructions

### Network requirements
The development environment requires outbound HTTPS access to:
- `pypi.org` and `files.pythonhosted.org` (Python package installs via pip/uv)
- `astral.sh` (uv installer, optional if uv installed via pip)
- `release-assets.githubusercontent.com` (GitHub release asset downloads, used by some install scripts)

If these are blocked, configure **Network Access** in your Cloud Agent settings to allow them.

### Dev environment setup
- **Install uv**: `pip install uv` (system-level, needed before anything else)
- **Install all dev deps**: `make dev` (runs `uv sync` with all required groups/extras)
- **Activate venv**: `. .venv/bin/activate` or prefix commands with `uv run`
- **Copy test secrets**: `cp tests/.dlt/dev.secrets.toml tests/.dlt/secrets.toml` (enables local-only destinations: duckdb, filesystem, postgres via containers)

### Key commands (see `Makefile` and `CONTRIBUTING.md` for full reference)
- **Lint**: `make lint` (mypy + ruff + flake8 + bandit + docstrings + lockfile check)
- **Format**: `make format` (black)
- **Test common (no external deps)**: `make test-common` or `make test-common-p` (parallel)
- **Test local loads (duckdb + filesystem)**: `make test-load-local`
- **Test with postgres containers**: `make start-test-containers` then `make test-load-local-postgres`
- **Build**: `uv build`

### Hello world verification
After `make dev`, verify the environment with a quick pipeline:
```python
import dlt

@dlt.resource
def hello():
    yield {"id": 1, "msg": "Hello from dlt!"}

pipeline = dlt.pipeline(pipeline_name="test", destination="filesystem", dataset_name="test")
import os; os.environ["DESTINATION__FILESYSTEM__BUCKET_URL"] = "/tmp/dlt_test_output"
info = pipeline.run(hello(), table_name="greetings")
print(info)
```
This uses the filesystem destination (no DB needed). For duckdb: `destination="duckdb"` (requires the `duckdb` extra).

### Gotchas
- dlt is a **library**, not a web app. There is no "server to start". Verify the environment by running `uv run python -c "import dlt; print(dlt.__version__)"` and then executing a small pipeline as above.
- Python 3.12 is fine for development. The project supports 3.9–3.14 but the venv uses whatever Python uv finds.
- Docker containers are **optional** unless you're testing specific destinations (postgres, clickhouse, weaviate, etc.). `make test-common` and `make test-load-local` (duckdb + filesystem) need no containers.
- The `uv.lock` file pins all dependency versions. After changing `pyproject.toml`, run `uv lock` to update it.
- C-extension dependencies (`orjson`, `pendulum`, `duckdb`) require pre-built wheels from PyPI. They cannot be installed from GitHub source alone.