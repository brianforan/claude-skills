# Backend: Python + FastAPI

**Why this stack:** async-first, type hints drive validation and OpenAPI docs automatically, and Pydantic gives typed config and I/O with little code. Django is the alternative when a project needs an admin UI, auth and ORM out of the box. It doesn't have a reference yet; write `backend/python-django.md` before the first Django project.

## Tooling

| Concern | Choice |
|---|---|
| Python | current stable (check), pinned in `.python-version` |
| Env + deps | `uv` (`uv init`, `uv add`, `uv run`). `pyproject.toml` + `uv.lock` committed |
| Server | `uvicorn` (`uv run uvicorn app.main:app --reload --port 8000`) |
| Config | `pydantic-settings` |
| DB | SQLAlchemy 2.x (typed `Mapped[...]` models), driver `psycopg` (v3), URL `postgresql+psycopg://` |
| Migrations | Alembic, autogenerate then review every migration by hand |
| Logging | `structlog` + `ecs-logging` (`ecs_logging.StructlogFormatter`) |
| Lint + format | `ruff check` + `ruff format` (config in `pyproject.toml`) |
| Types | `pyright` (basic mode to start) |
| Tests | `pytest` + `httpx` `AsyncClient` (or FastAPI `TestClient`) |

## Layout

```
api/
├── pyproject.toml
├── alembic.ini
├── migrations/               # alembic env + versions/
├── src/app/
│   ├── main.py               # create_app(): middleware, routers, exception handlers
│   ├── config.py             # Settings(BaseSettings), get_settings() cached
│   ├── logging.py            # configure_logging(settings): structlog + ECS
│   ├── db.py                 # engine, session factory, get_session dependency
│   ├── middleware.py         # request id + access log
│   ├── errors.py             # AppError + handlers → standard error JSON
│   ├── models/               # SQLAlchemy models (one module per aggregate)
│   ├── schemas/              # Pydantic request/response models
│   ├── services/             # business logic; no FastAPI imports here
│   └── routers/              # thin: parse → call service → return schema
│       └── health.py
└── tests/
    ├── conftest.py
    └── test_health.py
```

Rule of thumb: routers are thin, services hold logic, models know nothing about HTTP.

## Config

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=("../.env", ".env"), extra="ignore")
    database_url: PostgresDsn
    log_level: str = "info"
    log_file: Path | None = None
    service_name: str = "<project>-api"
    environment: str = "development"
```

## Logging setup

- `configure_logging()` runs once in `create_app()`. Processors: `merge_contextvars`, add log level, timestamp (ISO, UTC), `StackInfoRenderer`, `format_exc_info`, a redaction processor that drops or masks keys like `password`, `token`, `authorization`, `secret`, then `ecs_logging.StructlogFormatter()` last.
- Route stdlib logging (uvicorn, sqlalchemy) through the same output. Disable uvicorn's own access log (`--no-access-log` or `access_log=False`); the middleware writes one access line per request instead.
- Middleware: read or generate `X-Request-ID`, `structlog.contextvars.bind_contextvars(request_id=..., http_method=..., url_path=...)`, time the request, then log `request completed` with `http.response.status_code` and `event.duration`. Clear contextvars at the end.
- Add `service.name` and `service.environment` to every line.
- Outputs: stdout always, plus a file handler on `LOG_FILE` when set (NDJSON, one object per line).

## Errors

`AppError(code, message, status)` raised from services. Exception handlers turn it, `RequestValidationError` and uncaught exceptions into the standard error JSON (see conventions). Uncaught exceptions are logged at `error` with the stack trace; the response says `internal_error` and leaks nothing.

## Commands (README)

```
uv sync
uv run alembic upgrade head
uv run uvicorn app.main:app --reload --port 8000
uv run pytest
uv run ruff check . && uv run ruff format --check .
```

## Gotchas

- Use `src` layout and `uv run` so imports behave the same in tests and the app.
- Async SQLAlchemy needs `AsyncSession` and `create_async_engine` everywhere. Don't mix in sync sessions.
- Alembic autogenerate misses some changes (server defaults, enum changes, renames). Always read the generated file.
