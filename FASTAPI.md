# FastAPI coding guidelines

This document complements the general [Python guidelines](PYTHON.md) (linting, type hints,
logging, tracing, metrics, dependency management, ...) with conventions specific to our
[FastAPI](https://fastapi.tiangolo.com/) services. It is based on the boilerplate established in
[service-portal-state](https://github.com/swissgeo/service-portal-state), which every new FastAPI
service should use as a starting point.

- [1. Project structure](#1-project-structure)
  - [Folder responsibilities](#folder-responsibilities)
- [2. Settings](#2-settings)
- [3. Dependency injection](#3-dependency-injection)
- [4. Routers and endpoints](#4-routers-and-endpoints)
- [5. Schemas](#5-schemas)
- [6. OpenAPI documentation](#6-openapi-documentation)
- [7. Versioning](#7-versioning)
- [8. Docker image](#8-docker-image)
- [9. Testing](#9-testing)
- [10. Observability](#10-observability)

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
│   │   ├── __init__.py
│   │   ├── internal.py          # Non-public endpoints (e.g. kubernetes checker probe)
│   │   └── <feature>.py         # One module per feature/resource, each with its own APIRouter
│   ├── core/                    # App-wide config, business logic and utilities
│   │   ├── __init__.py
│   │   ├── db.py                # DB client/session dependency + (de)serialization helpers
│   │   ├── exceptions.py        # Centralized exception handlers
│   │   └── <feature>_service.py # Business/service layer, injected into routes via fastapi.Depends
│   ├── models/                  # DB models/ORM (e.g. SQLAlchemy ORM models, DynamoDB items)
│   │   ├── __init__.py
│   │   └── <feature>.py
│   ├── schemas/                 # Pydantic request/response models
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
├── .env.default                 # Default, non-secret env values, committed to the repo
├── .pre-commit-config.yaml      # Runs `make lint` before each commit
├── .python-version              # pinned python major and minor version for local development
├── Dockerfile                   # Multi-stage: base -> builder (uv) -> production (non-root)
├── docker-compose.yml           # Local dependencies (e.g. moto, otel-collector, jaeger)
├── Makefile                     # setup, lint, format, test, serve, dockerbuild, ...
├── pyproject.toml               # uv project + ruff + ty + pytest + coverage configuration
└── README.md
```

### Folder responsibilities

- **`app/api`**: HTTP layer only. Routers parse/validate input (via schemas), call into
  `app/core` services, and translate results/errors into responses. No business logic here.
- **`app/core`**: App-wide configuration, business/service logic, and infrastructure clients
  (database, external APIs). Services are plain classes exposed as dependencies (`XxxServiceDep`) that auto-inject service instances into route handlers, enabling easy swapping/mocking in tests.
- **`app/models`**: Database models/ORM definitions only — e.g. SQLAlchemy ORM models for a
  relational database, or the models representing DynamoDB items. No business logic.
- **`app/schemas`**: Pydantic models only — API request and response bodies. No business logic.
- **`app/middlewares`**: Cross-cutting request/response processing (hashing, auth, ...).
- **`app/tests`**: Mirrors the rest of `app/` with `test_*.py` files; shared fixtures live in
  `conftest.py`.

Keep one module per feature/resource in `api/`, `core/`, `models/`, and `schemas/` (e.g.
`state.py` in all four) rather than a single catch-all file, so the structure is easy to understand and intuitive.

## 2. Settings

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

## 3. Dependency injection

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

## 4. Routers and endpoints

- One `APIRouter` per feature module in `app/api`, tagged with a module-level constant
  (e.g. `STATE_TAG = "Application State"`) reused in `main.py`'s `openapi_tags`.
- Non-public endpoints (health/readiness probes, internal-only routes) go in `app/api/internal.py`
  under a dedicated `INTERNAL_TAG`, so they can be excluded from the public OpenAPI spec (see
  [OpenAPI documentation](#6-openapi-documentation)).
- Declare response models via the return type annotation (`-> SaveAppStateResponse`) and document
  error responses explicitly with `responses={400: {"model": ErrorResponse}, ...}`.
- Routers are registered in `app/main.py` with `app.include_router(...)`; `app/main.py` itself
  stays limited to app wiring (lifespan, middlewares, routers, instrumentation).
- Route handlers must not contain business logic; that belongs in the `app/core` service layer.

## 5. Schemas

- All request/response payloads are `pydantic.BaseModel`s in `app/schemas`, one module per
  feature. Database models/ORM definitions live in `app/models` instead (see
  [Project structure](#1-project-structure)).
- Use `Field(description=..., examples=[...])` on every field — this directly improves the
  generated OpenAPI docs.
- Sometimes JSON field names need to be in `camelCase` which conflicts with the Python dictionary key naming conventions (`snake_case`). To work around that, use `Field(alias=...)`, and endpoints returning them set `response_model_by_alias=True`.
- Share a single `ErrorResponse` schema (`app/schemas/errors.py`) across all error responses for
  consistency.

## 6. OpenAPI documentation

- FastAPI generates the OpenAPI schema from routes' type hints and schemas — keep every path
  operation fully typed so the generated docs stay accurate.
- Set `app.title`, `summary`, `description`, `contact` and `license_info` on the `FastAPI(...)`
  instance in `main.py`.
- If the service exposes internal-only endpoints, split the spec: a public
  `/openapi.json` (`/docs`, `/redoc`) excluding internal-tagged routes, and a separate
  `/internal/openapi.json` (`/internal/docs`, `/internal/redoc`) for the internal ones, as done in
  `app/openapi.py`.
- The OpenAPI specs are not published by default, you need to explicitly enable them through the `publish_openapi_spec` setting.

## 7. Versioning

`app/version.py` computes `__version__` from `git describe --tags` (falling back to the short
commit hash) for local/dev use, and is overwritten at Docker build time with the actual release
version:

```dockerfile
ARG VERSION=unknown
RUN echo "__version__ = '$VERSION'" > ${INSTALL_DIR}/app/version.py
```

Never hardcode a version string in application code — always read `app.version.__version__`.

## 8. Docker image

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

## 9. Testing

- `pytest` + `pytest-asyncio` + `pytest-cov` + `pytest-xdist`, run through `make test` /
  `make test-ci`.
- Shared fixtures live in `app/tests/conftest.py`:
   - A fixture of the FastAPI application with mocked settings
   - A fixture of the applications settings overriding environment-dependent values
- Mock external dependencies at the FastAPI dependency boundary with
  `app.dependency_overrides[get_xxx] = lambda: mock`, rather than monkeypatching internals.
- For AWS dependencies, prefer a per-test `ThreadedMotoServer` on a random port over mocking the
  library internals. This avoids having to maintain mocks for the AWS SDK and keeps tests closer
  to real-world usage, since they exercise the AWS SDK directly. For edge cases (e.g. simulating
  an aioboto3 failure), mocking library internals may still be necessary.

## 10. Observability

Structured logging, tracing and metrics conventions are covered in
[PYTHON.md](PYTHON.md#10-observability---logging) — for FastAPI services, additionally take into account:

- Initialize/shut down OpenTelemetry instrumentation from a dedicated `app/otel.py`, called from
  `main.py`'s startup and the `lifespan` shutdown block.
- Keep OTEL disabled by default for local dev (`make serve`) and use plain console logging
  instead, enabling OTEL only via a dedicated `.env.otel` + `make start-otel` local stack.
