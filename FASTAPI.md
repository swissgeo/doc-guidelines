# FastAPI coding guidelines

Complements the general [Python guidelines](PYTHON.md) (linting, type hints, logging, tracing,
metrics, dependency management, ...) with conventions specific to our
[FastAPI](https://fastapi.tiangolo.com/) services, based on the boilerplate established in
[service-portal-state](https://github.com/swissgeo/service-portal-state). Use it as the starting
point for every new FastAPI service.

- [1. Project structure](#1-project-structure)
- [2. Settings](#2-settings)
- [3. Dependency injection](#3-dependency-injection)
- [4. Main, routers and endpoints](#4-main-routers-and-endpoints)
- [5. OpenAPI documentation](#5-openapi-documentation)
- [6. Versioning](#6-versioning)
- [7. Docker image](#7-docker-image)
- [8. Testing](#8-testing)
- [9. Observability](#9-observability)

## 1. Project structure

Every FastAPI service should follow the same layout:

```text
service-name/
├── app/                         # Application source code
│   ├── __init__.py
│   ├── main.py                  # FastAPI() instance, lifespan, middlewares, routers registration
│   ├── settings.py              # pydantic-settings Settings + get_settings() dependency
│   ├── version.py               # __version__, overwritten at Docker build time
│   ├── openapi.py               # OpenAPI schema customization (e.g. public vs internal spec)
│   ├── otel.py                  # OpenTelemetry instrumentation setup/shutdown
│   ├── api/                     # HTTP layer: routers/controllers, no business logic
│   │   ├── README.md
│   │   ├── internal.py          # Non-public endpoints (e.g. kubernetes checker probe)
│   │   └── <feature>.py         # One module per feature/resource, each with its own APIRouter
│   ├── core/                    # App-wide config, business/service logic, infra clients
│   │   ├── README.md
│   │   ├── db.py                # DB client/session dependency + (de)serialization helpers
│   │   ├── exceptions.py        # Centralized exception handlers
│   │   └── <feature>_service.py # Service layer, injected into routes via Depends
│   ├── models/                  # DB models (e.g. SQLAlchemy ORM models)
│   │   ├── README.md
│   │   └── <feature>.py
│   ├── schemas/                 # Pydantic request/response models only
│   │   ├── README.md
│   │   ├── errors.py            # Shared ErrorResponse schema
│   │   └── <feature>.py
│   ├── middlewares/             # Cross-cutting request/response processing (hashing, auth, ...)
│   └── tests/
│       ├── conftest.py          # Shared fixtures (TestClient, dependency overrides, ...)
│       └── test_*.py
├── .dockerignore
├── .env.default                 # Default, non-secret env values, committed to the repo
├── .pre-commit-config.yaml      # Runs `make lint` before each commit
├── .python-version
├── Dockerfile                   # Multi-stage: base -> builder (uv) -> production (non-root)
├── docker-compose.yml           # Local dependencies (e.g. moto, otel-collector, jaeger)
├── Makefile                     # setup, lint, format, test, serve, dockerbuild, ...
├── pyproject.toml               # uv project + ruff + ty + pytest + coverage configuration
└── uv.lock
```

Keep one module per feature/resource across `api/`, `core/`, `models/` and `schemas/` (e.g.
`state.py` in all four) instead of catch-all files, so the layers stay easy to navigate together.

## 2. Settings

A single `pydantic-settings` `Settings` class in `app/settings.py`, loaded from environment
variables (or `.env` / `.env.default` files), exposed via `get_settings()` wrapped in
`@lru_cache` and a `SettingsDep = Annotated[Settings, Depends(get_settings)]` alias.

- `.env.default` (committed) holds non-secret defaults; `.env` (gitignored) overrides it locally;
  real deployments inject environment variables directly.
- List-typed settings must be parsed from comma-separated strings via a
  `field_validator(mode="before")`, since dotenv/Docker/pydantic-settings each quote JSON lists
  differently.

## 3. Dependency injection

Pair every injectable dependency (settings, DB client, service) with a `XxxDep` type alias next
to its provider function (e.g. `DynamoDBClientDep = Annotated[DynamoDBClient,
Depends(get_dynamodb_client)]`), and consume that alias in route/service signatures. This keeps
signatures short and makes dependencies easy to override in tests
(`app.dependency_overrides[get_dynamodb_client] = lambda: mock_client`).

## 4. Main, routers and endpoints

- `app/main.py` stays limited to app wiring (lifespan, middlewares, router registration,
  instrumentation) — no business logic.
- One `APIRouter` per feature module in `app/api`, tagged with a module-level constant reused in
  `main.py`'s `openapi_tags`.
- Non-public endpoints (health/readiness probes, ...) go in `app/api/internal.py` under a
  dedicated `INTERNAL_TAG`, excluded from the public OpenAPI spec (see
  [OpenAPI documentation](#5-openapi-documentation)).
- API endpoints must not contain business logic; that belongs in the `app/core/` packages.
- Declare response models via the return type annotation and document error responses explicitly
  with `responses={400: {"model": ErrorResponse}, ...}`.

## 5. OpenAPI documentation

- Keep every path operation fully typed — the OpenAPI schema is generated from route type hints
  and schemas.
- Set `title`, `summary`, `description`, `contact` and `license_info` on the `FastAPI(...)`
  instance in `main.py`.
- If the service exposes internal-only endpoints, split the spec: a public `/openapi.json`
  (`/docs`, `/redoc`) excluding internal-tagged routes, and a separate
  `/internal/openapi.json` (`/internal/docs`, `/internal/redoc`) for the internal ones.
- Only publish the spec when explicitly enabled (`publish_openapi_spec` setting), off by default.

## 6. Versioning

`app/version.py` computes `__version__` from `git describe --tags` (falling back to the short
commit hash) for local/dev use, and is overwritten at Docker build time with the actual release
version (`RUN echo "__version__ = '$VERSION'" > app/version.py`). Never hardcode a version string
elsewhere — always read `app.version.__version__`.

## 7. Docker image

A multi-stage `Dockerfile`:

1. **`base`**: minimal OS packages and a non-root user/group.
2. **`builder`**: installs dependencies with `uv sync --locked` (cache-mounted, dev dependencies
   excluded via `UV_NO_DEV=1`), then copies in `app/`.
3. **`production`**: copies the built `.venv` and `app/` from `builder`, overwrites `version.py`
   with the build `VERSION` arg, sets `git.*`/`author`/`version` image labels, runs as the
   non-root user, and starts the app (`fastapi run` or `uvicorn app.main:app`).

Only `app/` is copied into the final image — never tests, docs, or dev tooling.

## 8. Testing

- `pytest` + `pytest-asyncio` + `pytest-cov` + `pytest-xdist`, run through `make test` /
  `make test-ci`.
- Shared fixtures live in `app/tests/conftest.py`: a `client` fixture (`TestClient`) built from an
  `app` fixture that imports `app.main.app` *after* settings are mocked, plus a `settings`
  fixture overriding environment-dependent values.
- Mock external dependencies at the FastAPI dependency boundary
  (`app.dependency_overrides[get_xxx] = lambda: mock`) rather than monkeypatching internals.
- Prefer a real, ephemeral local backend over mocking library internals when practical — e.g. a
  per-test `moto` `ThreadedMotoServer` on a random port instead of a global mock decorator, for
  isolation under concurrent/async execution.

## 9. Observability

Structured logging, tracing and metrics conventions are covered in
[PYTHON.md](PYTHON.md#10-observability---logging). FastAPI services additionally initialize/shut
down OpenTelemetry instrumentation from a dedicated `app/otel.py` (called from `main.py`'s
startup and `lifespan` shutdown), and keep OTEL disabled by default for local dev (`make serve`),
enabling it only via a dedicated `.env.otel` + `make start-otel` local stack.
