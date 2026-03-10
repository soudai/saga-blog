# ADR 001: Application Architecture Baseline

Status: Accepted
Date: 2026-03-11

## Context

The repository is effectively empty, so the first implementation pass needs a clear baseline for the application shape, persistence strategy, and migration toolchain.

Earlier notes contained conflicting directions:
- PostgreSQL `16` vs `18`
- Alembic vs `golang-migrate`
- ORM-oriented backend notes vs the explicit requirement to use `2way-sql`

These conflicts need to be resolved before scaffolding begins.

## Decision

The project will use the following baseline:

- Repository layout is a monorepo with at least `frontend/`, `backend/`, `db/`, and `doc/`.
- Frontend stack remains `TypeScript`, `React`, `Vite`, `React Router`, `TanStack Query`, `React Hook Form`, `Zod`, and `shadcn/ui`.
- Backend stack remains `Python 3.12`, `FastAPI`, `Pydantic v2`, `pytest`, `Ruff`, and `mypy`.
- Database engine is `PostgreSQL 18`.
- Local orchestration is `Podman` with `podman compose`.
- Application queries use `2way-sql` as the primary database access mechanism.
- SQL files are stored as first-class source files under a dedicated SQL directory in the backend, not embedded inline across handlers.
- Schema migration uses `golang-migrate` with SQL migration files stored under `db/migrations/`.
- Persistence follows an immutable fact model: state changes are stored as append-only facts or events, and current state is derived by query.

## Consequences

- `Alembic` is not used for schema migration.
- ORM-managed domain state is not the source of truth. The backend may use a thin driver or transaction layer, but business queries must remain `2way-sql` driven.
- Table design needs both write-side fact tables and read-side queries that derive current article state, memberships, and notification targets.
- Migration files must be safe to run in order and must preserve the append-only model.
