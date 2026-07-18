# 2026-07-17 — Create docs/ONBOARDING.md

## Task
User requested a comprehensive `docs/ONBOARDING.md` synthesising all findings from the current session across 6 topics.

## Sources Read
- `AGENTS.md`, `.bob/rules-agent/AGENTS.md`, `.bob/rules-ask/AGENTS.md`, `.bob/rules-plan/AGENTS.md`
- `README.md`, `booking_system_frontend/README.md`
- `booking_system_backend/Dockerfile`, `requirements.txt`, `server.py`, `models.py`, `schemas.py`
- `booking_system_backend/services/user.py`, `flight.py`, `booking.py`
- `booking_system_backend/tests/conftest.py`, `test_services.py`, `test_rest.py`, `pytest.ini`
- `booking_system_frontend/package.json`, `tsconfig.app.json`, `src/services/api.ts`
- `.gitignore`, `start.sh`
- Glob/grep scans for docker-compose, terraform, .github/workflows, e2e tests, Dockerfile variants

## Output
Created `docs/ONBOARDING.md` (~430 lines) with 6 sections:
1. Application overview — purpose, structure, architecture diagram, directory table
2. Tech stack — backend + frontend tables, Stack at a Glance Mermaid diagram, design observations
3. Key components — responsibilities, component interaction map, booking lifecycle sequence diagram, cross-service contracts table
4. Unit test coverage — infrastructure detail, test tables for test_services.py and test_rest.py, coverage gaps table
5. E2E test coverage — confirmed none exist; listed what would be required; flows not covered
6. Deployment model — confirmed no CI/CD/IaC; documented what exists (1 Dockerfile, start.sh); constraints table

## Key Findings Captured
- `booking_system_inventory_hold_service` does not exist (noted explicitly)
- No E2E tests exist anywhere in the repo
- Frontend has zero test infrastructure
- `fly.toml` is gitignored — historical Fly.io usage, no longer active
- Frontend Dockerfile exists only as a README code block, not as a real file
- MCP tool wrappers are untested
- `seed()` patched to no-op in all tests — never verified
