---
stack: python
priority: 100
aspects: [backend]
detect:
  any:
    - file_exists: pyproject.toml
    - file_exists: requirements.txt
    - file_exists: setup.py
    - file_exists: Pipfile
---

# Python Stack Profile (backend)

Registers plain Python projects with the SDLC pipeline. Auto-detected by the presence of `pyproject.toml`, `requirements.txt`, `setup.py`, or `Pipfile`.

This plugin owns the **backend** aspect for Python projects that are **not** a recognized web framework — libraries, CLI tools, scripts, data pipelines, microservices. When `django-plugin`, `fastapi-plugin`, or `flask-plugin` match (priority 150), they win the backend aspect instead and this plugin plays no role. No frontend aspect is owned.

## Agents per phase

```yaml
business_analysis: business-analyst   # core agent
development:
  backend: python-architect           # owned by this plugin
qa: qa-engineer                       # core agent
security: security-analyst            # core agent
documentation: document-writer        # core agent
```

Note: this plugin does NOT declare `development.frontend` and does NOT have an extra database phase. Framework plugins (django/fastapi/flask) at priority 150 supersede this profile entirely when present.

## Convention skills to apply

- python-foundation:python-conventions
- python-foundation:python-tooling
- python-foundation:pytest-testing
- python-plugin:python-app-conventions

## Phase prompts injection

For development phase (backend aspect), inject:
> You are working on the **backend** aspect of a **plain Python** project (library, CLI tool, script, data pipeline, or microservice without a web framework). Your scope:
> - Application / library structure, module design, CLI entry points, core business logic, data processing pipelines, external API clients, configuration management, and packaging.
>
> Start by reading `pyproject.toml` (or `requirements.txt` / `setup.py`) to understand the project structure, installed packages, package manager in use (poetry.lock → Poetry, uv.lock → uv, else pip), and entry points.
>
> Apply `python-foundation:python-conventions`:
> - **Type hints:** all functions annotated. Use `from __future__ import annotations` for forward references. Run `mypy` in the mode the project configures (strict or default).
> - **Dataclasses / enums:** prefer `@dataclass` or `@dataclass(frozen=True)` for plain data containers. Use `enum.Enum` or `enum.StrEnum` for categorical values.
> - **match/case:** use structural pattern matching (Python 3.10+) where it simplifies branching on data shapes.
> - **pathlib:** use `pathlib.Path` for all file system operations — never `os.path.join`.
> - **Context managers:** use `with` for all resource management (files, DB cursors, HTTP sessions, locks).
>
> Apply `python-foundation:python-tooling`:
> - Detect the package manager from the lockfile: `poetry.lock` → Poetry, `uv.lock` → uv, else pip.
> - Run `ruff format .` (or `ruff check --fix . && ruff format .`) after writing — never hand-tune whitespace.
> - Run `mypy src/` or `mypy .` as advisory (report failures but do not block on projects that have no mypy config).
>
> Apply `python-plugin:python-app-conventions`:
> - Respect the project's existing layout: `src/<package>/` preferred; flat layout acceptable for small projects.
> - Configuration from environment via `pydantic-settings` or `python-decouple`. Never `os.environ.get()` scattered inline.
> - Structured logging via `logging.getLogger(__name__)` — never `print()` for diagnostics (except CLI output in `__main__.py` / `cli.py`).
> - CLI via `argparse`, `click`, or `typer` — match the tool already in use; register entry points in `[project.scripts]`.
>
> After writing code, run:
> - `ruff format .` — auto-formats; do not iterate on style manually.
> - `ruff check --fix .` — auto-fixes lint violations where safe.
> - `mypy src/` or `mypy .` — advisory; report result.
> - `python -c "import <package>"` — smoke-test for import errors.
> - `python -m pytest` — run the full test suite (advisory; qa-engineer fills in missing tests).

For qa phase, inject:
> Apply `python-foundation:pytest-testing`:
> - **Unit tests:** pure functions with no I/O — no mocking needed. Cover happy path, edge cases, and error conditions.
> - **Integration tests:** involve real I/O (DB, filesystem, external API). Use pytest fixtures to set up and tear down state. Parametrize for multiple inputs with `@pytest.mark.parametrize`.
> - **Mock external I/O:** HTTP via `responses` or `httpx` mock transport, filesystem via `monkeypatch` and `tmp_path`, time via `monkeypatch.setattr(datetime, 'now', ...)`.
> - **Coverage:** run `pytest --cov=src --cov-report=term-missing` (or `pytest --cov=. --cov-report=term-missing` for flat layouts). Aim for ≥ 80 % coverage on new code.
>
> Run: `pytest` or `python -m pytest`.

For security phase, inject:
> Check Python language-level rules (from `python-foundation`) plus application-level risks:
> - **Code execution:** `eval()` / `exec()` with user-controlled input — flag any occurrence. Prefer AST parsing (`ast.literal_eval`) for safe evaluation of literals.
> - **Subprocess injection:** `subprocess.*` with `shell=True` and concatenated user input — replace with list-form invocation (`subprocess.run(["cmd", arg])`) without `shell=True`.
> - **Unsafe deserialization:** `pickle.loads()` on untrusted data — replace with `json`, `msgpack`, or `safetensors`. `yaml.load()` without `Loader=yaml.SafeLoader` — always use `yaml.safe_load()`.
> - **Hardcoded secrets:** scan for API keys/tokens in source (variable names: `api_key`, `token`, `secret`, `password`, `auth`; also URL strings containing credentials). All secrets must come from environment variables or a secrets manager.
> - **Insecure random:** `random.random()` / `random.randint()` for security-sensitive use (tokens, IDs, nonces) — use `secrets.token_urlsafe()` / `secrets.token_bytes()` / `secrets.randbelow()` instead.
> - **Path traversal:** `open()` / `pathlib.Path()` with user-supplied paths — resolve with `Path.resolve()` and assert `resolved.is_relative_to(base_dir)`.
> - **Temp file race:** `tempfile.mktemp()` — replace with `tempfile.mkstemp()` or `tempfile.NamedTemporaryFile()`.
> - **SQL injection:** f-string or `.format()` interpolation into SQL queries — use parameterized queries with the DB driver's `?` / `%s` / `$1` placeholders.

## Post-pipeline checks

```
ruff check . --no-fix
ruff format --check .
mypy . (advisory — failures reported but do not retry)
python -m pytest
```

These run after the documentation phase. Advisory — failures are reported but do not retry.

## MCP integration

Python has no standard MCP server equivalent. Agents use the Python CLI via Bash (or `docker compose exec -T app python ...` in Dockerized setups) for running scripts, executing the package, and running the test suite. The pipeline runs fully without any MCP server.
