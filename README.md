# Task Manager

A task-management system for HMCTS caseworkers, built for the DTS Developer
technical test. Caseworkers can create, view, update and delete tasks (title,
optional description, status and due date/time) through an accessible GOV.UK
web interface backed by a REST API and a relational database.

The system is split into two independently deployable services, mirroring the
way HMCTS runs its backend and frontend applications as separate repositories:

```
                  ┌──────────────────────────────────────────────┐
  Caseworker ───► │  frontend  (Node / TypeScript / Express)       │
  (browser)       │  GOV.UK Design System + Nunjucks, CSRF-protected│
                  └───────────────┬────────────────────────────────┘
                                  │  REST over HTTP (JSON)
                                  ▼
                  ┌──────────────────────────────────────────────┐
                  │  backend   (Java 21 / Spring Boot)             │
                  │  Bean Validation, RFC 7807 errors, OpenAPI     │
                  └───────────────┬────────────────────────────────┘
                                  │  JPA / parameterised SQL
                                  ▼
                  ┌──────────────────────────────────────────────┐
                  │  database  (PostgreSQL, schema owned by Flyway)│
                  └──────────────────────────────────────────────┘
```

| Service               | Stack                                                        | Port  |
|-----------------------|--------------------------------------------------------------|-------|
| [`backend`](./backend)  | Java 21, Spring Boot 3.5, Spring Data JPA, Flyway, springdoc | 4000  |
| [`frontend`](./frontend)| Node, TypeScript, Express, Nunjucks, GOV.UK Frontend         | 3100  |
| `database`            | PostgreSQL 16                                                 | (internal) |

## Run the whole system

The only prerequisite is Docker. From this directory:

```bash
docker compose up --build
```

This builds and starts all three services in the correct order (the backend
waits for the database, the frontend waits for the backend's health check).
Then open:

- **Frontend** — http://localhost:3100
- **Backend API** — http://localhost:4000/tasks
- **Swagger UI** — http://localhost:4000/swagger-ui/index.html
- **Health** — http://localhost:4000/health and http://localhost:3100/health

Stop everything with `Ctrl+C`, or `docker compose down` (add `-v` to also drop
the database volume).

Ports and database credentials can be overridden via a `.env` file — see
[`.env.example`](./.env.example). The database is not published to the host; it
is reachable only by the backend over the internal Docker network.

## Running a service on its own

Each repository is self-contained and can be developed without the other — see
its README for details:

- [`backend/README.md`](./backend/README.md) — `./gradlew bootRun` against a
  local Postgres (`docker compose up -d database`).
- [`frontend/README.md`](./frontend/README.md) — `yarn start:dev` against a
  running backend.

## Testing

| Layer | Where | Command |
|-------|-------|---------|
| Backend unit (service + web layer) | `backend` | `./gradlew test` |
| Backend integration (real Postgres via Testcontainers) | `backend` | `./gradlew integration` |
| Frontend unit / routes / accessibility | `frontend` | `yarn test:unit`, `yarn test:routes`, `yarn test:a11y` |
| End-to-end browser journey (create → view → edit → re-status → delete) | `frontend` | `yarn test:functional` |

Both repositories run their checks in GitHub Actions on every push and pull
request, and are scanned with CodeQL.

### End-to-end smoke test

`frontend/src/test/functional` contains a CodeceptJS + Playwright scenario that
drives the full task lifecycle through the real UI against the real backend.
With the stack running (`docker compose up`):

```bash
cd frontend
yarn playwright install chromium        # first run only
TEST_URL=http://localhost:3100 yarn test:functional
```

Run it on the version of Node pinned in `frontend/.nvmrc` (`nvm use`); the
browser-teardown step is only clean on the supported Node versions.

## Design decisions and scope

- **Two services, not a monorepo** — matches how HMCTS structures its products
  and lets each service build, test, deploy and scale independently.
- **The frontend holds no state.** All persistence is delegated to the backend;
  the browser only ever issues `GET`/`POST`, and the server uses the full set of
  REST verbs.
- **Configuration is environment-driven** (database connection, `API_URL`,
  ports) so the same images run locally and in any deployed environment.

Deliberately out of scope for this exercise, but how each would be approached in
production:

- **Authentication / authorisation** — IDAM (OAuth2/OIDC) at the edge, with
  per-user task ownership and non-enumerable identifiers to avoid IDOR.
- **Infrastructure as code & hosting** — Terraform onto Azure (AKS), with the
  two images deployed as separate Helm releases.
- **Observability** — structured logs plus App Insights / Dynatrace dashboards
  and alerting.

## Security

- All input is validated at the API boundary (Bean Validation) and again in the
  frontend before submission; the status column also has a database `CHECK`
  constraint.
- Every state-changing form carries a CSRF token; a bad token is rejected.
- Persistence uses parameterised JPA queries, so user input is never
  concatenated into SQL.
- Error responses never leak stack traces or internal detail (RFC 7807
  `application/problem+json` from the API; friendly GOV.UK error pages in the UI).
- No secrets are committed; the database is not exposed outside the Docker
  network.
