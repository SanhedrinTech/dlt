# dlt principles
- This is a mature library used by thousands of the best engineers in the world
- dlt is a library, not a platform. users add it to their code. we respect existing workflows and other libraries they use
- no black boxes: clean, minimal Pythonic interfaces, human readable file formats, no side effects, documentation
- multiply - don't add: we do more work here so our users do less.

# dlt repo
- we use `uv` and `uv run`. look in @Makefile
- we branch from `devel` for feature branches

## Cursor Cloud specific instructions

### Quick reference
- **Dev setup**: `make install-uv && make dev` (installs uv + all dev deps via `uv sync`)
- **Activate venv**: `source .venv/bin/activate` or prefix commands with `uv run`
- **Lint**: `make lint` (mypy, ruff, flake8, bandit, docstrings, lockfile)
- **Format**: `make format` (black)
- **Test (no external deps)**: `make test-common` or `make test-common-p` (parallel)
- **Test local loads (DuckDB + filesystem)**: `make test-load-local`
- **Test with Postgres**: `make start-test-containers && make test-load-local-postgres`
- See `Makefile` and `CONTRIBUTING.md` for full details.

### Gotchas
- `uv` must be on `PATH` after install; the update script handles this via `curl ... | sh` which puts it in `~/.local/bin`. If `uv` isn't found, ensure `~/.local/bin` is on `PATH`.
- `make dev` calls `uv sync --all-extras --no-extra hub --group dev ...` which creates `.venv` in the workspace root. All `make` targets use `uv run` so activating the venv is optional.
- `orjson` is a required dependency (hard import in `dlt/destinations/impl/filesystem/filesystem.py`). It's a Rust extension installed via `uv sync`; it cannot be easily replaced.
- The test suite uses `ACTIVE_DESTINATIONS` and `ALL_FILESYSTEM_DRIVERS` env vars to control which destinations and filesystem drivers are tested. Defaults vary per make target.
- Tests that need Docker containers (Postgres, ClickHouse, etc.) require `make start-test-containers` first. Docker must be installed and running.
- `tests/.dlt/dev.secrets.toml` contains template credentials for local testing; copy it to `tests/.dlt/secrets.toml` for use.