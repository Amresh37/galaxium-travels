# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Overview

Galaxium Travels — a space-flight booking system with two sub-projects:
- `booking_system_backend/` — Python FastAPI + FastMCP server (port 8080)
- `booking_system_frontend/` — React 19 + TypeScript + Vite + Tailwind CSS frontend

## Backend Commands

All commands must be run from `booking_system_backend/`:

```bash
# Run server
python server.py

# Run all tests
pytest

# Run a single test file
pytest tests/test_services.py
pytest tests/test_rest.py

# Run a single test by name
pytest tests/test_services.py::TestBookingService::test_book_flight_success
```

## Frontend Commands

All commands must be run from `booking_system_frontend/`:

```bash
npm run dev       # dev server
npm run build     # tsc -b && vite build
npm run lint      # eslint
```

## Critical Architecture Notes

**Dual protocol server**: `server.py` exposes the same business logic via both REST (FastAPI) and MCP (FastMCP). The MCP server is mounted at `/mcp` inside the FastAPI app. The `mcp` instance **must be created before** the FastAPI `app` so that lifespans combine correctly.

**MCP tools manage their own DB sessions** using `SessionLocal()` directly (not `get_db` dependency), because they are not request-scoped. REST endpoints use `Depends(get_db)`.

**`seed()` wipes and re-seeds the DB on every server startup** — the database is SQLite at `booking.db` in the backend directory.

**Error handling pattern**: Service functions return `ErrorResponse` (not raise exceptions) on failure. REST endpoints return HTTP 200 with the `ErrorResponse` payload. MCP tool wrappers convert `ErrorResponse` to raised `Exception`.

**`book_flight` validates both `user_id` AND `name`** — name must match the registered user's name. Passing only `user_id` is not enough.

## Backend Code Style

- Imports are local/relative (e.g. `from models import ...`, not `from booking_system_backend.models import ...`)
- `conftest.py` adds the parent directory to `sys.path` — test files also manually do `sys.path.insert(0, str(Path(__file__).parent.parent))`
- Pydantic schemas use `class Config: from_attributes = True` for ORM model serialization
- Use `BookingOut.model_validate(obj)` (not `.from_orm()`) to serialize SQLAlchemy models
- Booking statuses are string literals: `"booked"`, `"cancelled"`, `"completed"`
- Datetimes stored as ISO 8601 strings in SQLite, not datetime columns

## Frontend Code Style

- All API types live in `src/types/index.ts` — mirroring backend Pydantic schemas
- API calls go through the axios instance in `src/services/api.ts`; use `isErrorResponse()` helper to discriminate union returns
- Frontend base URL configured via `VITE_API_URL` env var (see `.env.example`); defaults to `http://localhost:8080`
- `verbatimModuleSyntax` is enabled — use `import type { ... }` for type-only imports
- `noUnusedLocals` and `noUnusedParameters` are enforced by TypeScript

## Testing Specifics

- Tests use an **in-memory SQLite** database with `StaticPool` — each test function gets a fresh schema
- The `client` fixture patches `server.SessionLocal` and `db.SessionLocal` via `monkeypatch` to route MCP tool DB calls to the test session
- `seed()` is patched to a no-op during tests
- **pytest must be run from `booking_system_backend/`** — `pytest.ini` sets `testpaths = tests`
