# FastAPI coding guidelines

This document complements the general [Python guidelines](PYTHON.md) (linting, type hints,
logging, tracing, metrics, dependency management, ...) with conventions specific to our
[FastAPI](https://fastapi.tiangolo.com/) services. It is based on the boilerplate established in
[service-portal-state](https://github.com/swissgeo/service-portal-state), which every new FastAPI
service should use as a starting point.

- [1. Project structure](#1-project-structure)
  - [Folder responsibilities](#folder-responsibilities)
  - [Self-documenting folders](#self-documenting-folders)
- [2. Tooling](#2-tooling)
- [3. Settings](#3-settings)
- [4. Dependency injection](#4-dependency-injection)
- [5. Routers and endpoints](#5-routers-and-endpoints)
- [6. Schemas](#6-schemas)
- [7. Centralized error handling](#7-centralized-error-handling)
- [8. OpenAPI documentation](#8-openapi-documentation)
- [9. Versioning](#9-versioning)
- [10. Docker image](#10-docker-image)
- [11. Testing](#11-testing)
- [12. Observability](#12-observability)

## 1. Project structure

Every FastAPI service should follow the same layout, so that switching between services stays
predictable:

```text
service-name/
├── app/                         # Application source code
│   ├── __init__.py
│   ├── main.py                  # FastAPI() instance, lifespan, middlewares, routers registration
│   ├── settings.py              # pydantic-settings Settings + get_settings() dependency
│   ├── version.py               # __version__, overwritten at Docker build time
│   ├── openapi.py               # OpenAPI schema customization (e.g. public vs internal spec)
│   ├── otel.py                  # OpenTelemetry instrumentation setup/shutdown
│   ├── api/                     # API routes and controllers
│   │   ├── README.md
│   │   ├── __init__.py
│   │   ├── internal.py          # Non-public endpoints (e.g. kubernetes checker probe)
│   │   └── <feature>.py         # One module per feature/resource, each with its own APIRouter
│   ├── core/                    # App-wide config, business logic and utilities
│   │   ├── README.md
│   │   ├── __init__.py
│   │   ├── db.py                # DB client dependency + (de)serialization helpers
│   │   ├── exceptions.py        # Centralized exception handlers
│   │   └── <feature>_service.py # Business/service layer, injected into routes via Depends
│   ├── schemas/                 # Pydantic request/response/DB models
│   │   ├── README.md
│   │   ├── __init__.py
│   │   ├── errors.py            # Shared ErrorResponse schema
│   │   └── <feature>.py
│   ├── middlewares/             # Custom Starlette/FastAPI middlewares
│   │   ├── __init__.py
│   │   └── <name>.py
│   └── tests/
│       ├── __init__.py
│       ├── conftest.py          # Shared fixtures (TestClient, dependency overrides, ...)
│       └── test_*.py
├── .dockerignore
├── .env.default                 # Default, non-secret env values, committed to the repo
├── .gitignore
├── .pre-commit-config.yaml      # Runs `make lint` before each commit
├── .python-version
├── .vscode/settings.json        # ruff as formatter on save, pytest integration
├── Dockerfile                   # Multi-stage: base -> builder (uv) -> production (non-root)
├── docker-compose.yml           # Local dependencies (e.g. moto, otel-collector, jaeger)
├── Makefile                     # setup, lint, format, test, serve, dockerbuild, ...
├── pyproject.toml               # uv project + ruff + ty + pytest + coverage configuration
├── README.md
└── uv.lock
```

### Folder responsibilities

- **`app/api`**: HTTP layer only. Routers parse/validate input (via schemas), call into
  `app/core` services, and translate results/errors into responses. No business logic here.
- **`app/core`**: App-wide configuration, business/service logic, and infrastructure clients
  (database, external APIs). Services are plain classes/functions exposed as dependencies
  (`XxxServiceDep`) so they can be swapped/mocked in tests.
- **`app/schemas`**: Pydantic models only — request bodies, response bodies, and
  database item models. No business logic.
- **`app/middlewares`**: Cross-cutting request/response processing (hashing, auth, ...).
- **`app/tests`**: Mirrors the rest of `app/` with `test_*.py` files; shared fixtures live in
  `conftest.py`.

Keep one module per feature/resource in `api/`, `core/` and `schemas/` (e.g. `state.py` in all
three) rather than a single catch-all file, so the three layers stay easy to navigate together.

### Self-documenting folders

Each top-level folder under `app/` (`api/`, `core/`, `schemas/`) has a short `README.md`
describing its purpose, for example `app/core/README.md`:

```markdown
# Application core

App-wide config, settings, security utilities, ...
```

Add this `README.md` to any new folder you introduce under `app/`, so the structure remains
self-explanatory without needing this guideline open.

## 2. Tooling

- **[uv](https://docs.astral.sh/uv/)** manages the virtualenv and dependencies (`uv.lock`
  committed). Python version is pinned in `.python-version`.
- **[ruff](https://docs.astral.sh/ruff/)** is used for both linting and formatting (see
  [PYTHON.md](PYTHON.md#1-linting--auto-formatting) for the base rule set).
- **[ty](https://github.com/astral-sh/ty)** is used for static type checking.
- **pre-commit** runs `make lint` before every commit (`.pre-commit-config.yaml`); it can be
  bypassed with `git commit --no-verify` or selectively with `SKIP=lint git commit ...` when
  needed.
- A **`Makefile`** is the single entrypoint for common tasks and should expose at least:
  `setup`, `serve`, `format`, `lint`, `test`, `test-ci`, `dockerbuild`, `dockerrun`,
  `dockerpush`, `help` (self-documented via `##` comments, see `make help`).

## 3. Settings

Configuration is a single `pydantic-settings` `Settings` class in `app/settings.py`, loaded from
environment variables (or `.env` / `.env.default` files):

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=(".env", ".env.default"),
        env_file_encoding="utf-8",
        enable_decoding=False,
        extra="ignore",
    )

    aws_region: str
    cors_origins: list[str] = []
    # ...


@lru_cache
def get_settings() -> Settings:
    return Settings()


SettingsDep = Annotated[Settings, Depends(get_settings)]
```

- Wrap `get_settings()` in `@lru_cache` so settings are parsed once per process, yet stay
  overridable in tests via FastAPI's `app.dependency_overrides`.
- `.env.default` holds non-secret defaults and is committed; `.env` (gitignored) overrides it
  locally; real deployments inject environment variables directly.
- List-typed settings coming from env vars/`.env` files must be comma-separated strings parsed
  with a `field_validator(mode="before")`, since dotenv/Docker/pydantic-settings each quote JSON
  lists differently.

## 4. Dependency injection

Every injectable dependency (settings, DB client, service) is paired with a `XxxDep` type alias
next to its provider function, and consumed by that alias in route/service signatures:

```python
DynamoDBClientDep = Annotated[DynamoDBClient, Depends(get_dynamodb_client)]

async def get_app_state(state_id: StateId, app: StateServiceDep) -> GetAppStateResponse:
    ...
```

This keeps signatures short, makes dependencies easy to override in tests
(`app.dependency_overrides[get_dynamodb_client] = lambda: mock_client`), and avoids re-wiring
`Depends(...)` at every call site.

## 5. Routers and endpoints

- One `APIRouter` per feature module in `app/api`, tagged with a module-level constant
  (`STATE_TAG = "Application State"`) reused in `main.py`'s `openapi_tags`.
- Non-public endpoints (health/readiness probes, internal-only routes) go in `app/api/internal.py`
  under a dedicated `INTERNAL_TAG`, so they can be excluded from the public OpenAPI spec (see
  [OpenAPI documentation](#8-openapi-documentation)).
- Declare response models via the return type annotation (`-> SaveAppStateResponse`) and document
  error responses explicitly with `responses={400: {"model": ErrorResponse}, ...}`.
- Routers are registered in `app/main.py` with `app.include_router(...)`; `app/main.py` itself
  stays limited to app wiring (lifespan, middlewares, routers, instrumentation) — no business
  logic.

## 6. Schemas

- All request/response/DB payloads are `pydantic.BaseModel`s in `app/schemas`, one module per
  feature.
- Use `Field(description=..., examples=[...])` on every field — this directly improves the
  generated OpenAPI docs.
- Python code stays `snake_case`; client-facing JSON fields that need `camelCase` use
  `Field(alias=...)`, and endpoints returning them set `response_model_by_alias=True`.
- Share a single `ErrorResponse` schema (`app/schemas/errors.py`) across all error responses for
  consistency.

## 7. Centralized error handling

Register exception handlers once, in `app/core/exceptions.py`, and normalize every error path
(validation errors, `HTTPException`, unexpected exceptions) to the same `ErrorResponse` schema:

```python
def register_exception_handlers(app: FastAPI) -> None:
    app.add_exception_handler(Exception, unified_exception_handler)
    app.add_exception_handler(RequestValidationError, unified_exception_handler)
    app.add_exception_handler(HTTPException, unified_exception_handler)
```

Routes should raise `HTTPException` (or a domain exception mapped to one) rather than building
`JSONResponse`s by hand.

## 8. OpenAPI documentation

- FastAPI generates the OpenAPI schema from routes' type hints and schemas — keep every path
  operation fully typed so the generated docs stay accurate.
- Set `app.title`, `summary`, `description`, `contact` and `license_info` on the `FastAPI(...)`
  instance in `main.py`.
- If the service exposes internal-only endpoints, split the spec: a public
  `/openapi.json` (`/docs`, `/redoc`) excluding internal-tagged routes, and a separate
  `/internal/openapi.json` (`/internal/docs`, `/internal/redoc`) for the internal ones, as done in
  `app/openapi.py`.
- Only publish the spec when explicitly enabled (`publish_openapi_spec` setting), off by default.

## 9. Versioning

`app/version.py` computes `__version__` from `git describe --tags` (falling back to the short
commit hash) for local/dev use, and is overwritten at Docker build time with the actual release
version:

```dockerfile
ARG VERSION=unknown
RUN echo "__version__ = '$VERSION'" > ${INSTALL_DIR}/app/version.py
```

Never hardcode a version string in application code — always read `app.version.__version__`.

## 10. Docker image

Use a multi-stage `Dockerfile`:

1. **`base`**: minimal OS packages and a non-root user/group.
2. **`builder`**: installs dependencies with `uv sync --locked` (using
   `--mount=type=cache` for the uv cache and bind-mounting `pyproject.toml`/`uv.lock`), then
   copies in `app/`.
3. **`production`**: copies the built `.venv` and `app/` from `builder`, overwrites
   `version.py` with the build `VERSION` arg, sets `git.*`/`author`/`version` image labels, runs
   as the non-root user, and starts the app (`fastapi run` or `uvicorn app.main:app`).

Only `app/` (never tests, docs, or dev tooling) is copied into the final image; dev dependencies
are excluded via `UV_NO_DEV=1`.

## 11. Testing

- `pytest` + `pytest-asyncio` + `pytest-cov` + `pytest-xdist`, run through `make test` /
  `make test-ci`.
- Shared fixtures live in `app/tests/conftest.py`: a `client` fixture (`TestClient`) built from an
  `app` fixture that imports `app.main.app` *after* settings are mocked, plus a `settings` fixture
  overriding environment-dependent values (endpoints, region, ...).
- Mock external dependencies at the FastAPI dependency boundary with
  `app.dependency_overrides[get_xxx] = lambda: mock`, rather than monkeypatching internals.
- Prefer exercising a real (but local/ephemeral) backend over mocking library internals when
  practical — e.g. a dedicated per-test server (such as a `moto` `ThreadedMotoServer` on a random
  port) instead of a global mock decorator, for proper test isolation under concurrent/async
  execution.

## 12. Observability

Structured logging, tracing and metrics conventions are covered in
[PYTHON.md](PYTHON.md#10-observability---logging) — FastAPI services additionally:

- Initialize/shut down OpenTelemetry instrumentation from a dedicated `app/otel.py`, called from
  `main.py`'s startup and the `lifespan` shutdown block.
- Keep OTEL disabled by default for local dev (`make serve`) and use plain console logging
  instead, enabling OTEL only via a dedicated `.env.otel` + `make start-otel` local stack.
