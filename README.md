# Primary Ticket Outlet Monorepo

This monorepo hosts the Primary Ticket Outlet MVP: a Spring Boot backend, a Vite/React frontend, an nginx wrapper, and a lightweight payment stub service. The stack is container-friendly, fully tested, and ships with deterministic database seeds for local and CI environments.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Layout](#project-layout)
3. [Local Development Workflow](#local-development-workflow)
4. [Backend](#backend)
5. [Database Migrations & Seeding](#database-migrations--seeding)
6. [Frontend](#frontend)
7. [Testing & Coverage](#testing--coverage)
8. [End-to-End Testing](#end-to-end-testing)
9. [Docker Orchestration](#docker-orchestration)
10. [Simulating a Fresh Deployment](#simulating-a-fresh-deployment)
11. [Troubleshooting](#troubleshooting)
12. [Additional Resources](#additional-resources)

---

## Architecture Overview

| Component | Stack                                                   | Responsibilities |
|-----------|---------------------------------------------------------|------------------|
| Backend | Spring Boot 4 · Java 25 (Temurin) · PostgreSQL · Flyway | REST API with controller/service/repository layering, JWT-like token verification, role & venue management, ticket purchase orchestration |
| Frontend | React 19 · React Router 7 · Vite · MUI                  | SPA with mock SSO login, role-based dashboards (attendee/manager/admin), event browsing, ticket purchasing, administration dashboards |
| Infra | Docker Compose · nginx proxy · Node payment stub        | Local orchestration of Postgres, payment microservice mock, backend, and compiled frontend |

Backend highlights:
- Controller → Service → Repository separation with DTO/model layers under `com.tickets.backend`.
- Models live in `model/` with Lombok (`@Data`, `@Builder`, `@RequiredArgsConstructor`) to keep entities concise.
- Repeatable Flyway migrations seed roles, users, venues, and events for every deployment.
- Payment integration talks to the local stub (Node/Express) via `PAYMENT_BASE_URL`.

Frontend highlights:
- Feature-oriented folders (`features/auth`, `features/dashboard`, `features/navigation`) encapsulate UI, hooks, and tests.
- `useAuthSession`, `useEvents`, and `useManagerVenues` custom hooks separate API/data logic from presentational components.
- React Router drives navigation between `/login`, `/`, `/manager`, and `/admin`, with `DashboardLayout` providing the shared shell and role switcher.
- Playwright tests cover mock login flows and role-based dashboards.

---

## Project Layout

```
project/
├── backend/                  # Spring Boot application (Gradle)
│   ├── src/main/java/com/tickets/backend/
│   │   ├── controller/       # REST controllers
│   │   ├── service/          # Business logic & facades
│   │   ├── repository/       # Spring Data JPA repositories
│   │   ├── dto/              # API payloads
│   │   ├── model/            # JPA entities
│   │   ├── config/util/...   # Support modules
│   ├── src/main/resources/
│   │   └── db/migration/     # V1 schema + repeatable data seeds
│   ├── src/test/java/...     # Unit & controller/service tests
│   ├── build.gradle          # Backend build config
│   └── Dockerfile            # Multi-stage build (Temurin 25)
├── frontend/                 # Vite/React SPA
│   ├── src/
│   │   ├── app/              # App router & providers
│   │   ├── api/              # REST helpers
│   │   ├── features/
│   │   │   ├── auth/         # Auth context, login page, hooks
│   │   │   ├── dashboard/    # Layout, dashboards, domain hooks
│   │   │   └── navigation/   # Role switcher UI
│   │   └── main.jsx          # entry point
│   ├── tests/                # Playwright E2E specs
│   ├── coverage/             # Vitest & Playwright output
│   ├── vite.config.js        # Vite + Vitest settings
│   ├── playwright.config.ts  # Playwright settings
│   └── Dockerfile            # Built via nginx multi-stage build
├── nginx/                    # nginx Dockerfile + config for static hosting
├── payment/                  # Node/Express payment stub (npm based)
├── docker-compose.yml        # Full stack orchestration (postgres/payment/backend/nginx)
├── readme_mvp.md             # Original MVP product brief
└── README.md                 # This guide
```

---

## Local Development Workflow

**Prerequisites**
- Java 25 (Temurin) – the Gradle toolchain will enforce this.
- Node.js 18+ and npm.
- Docker & Docker Compose (for full-stack runs and integration tests).

**Initial setup**
```bash
git clone <repo-url>
cd project

# Frontend dependencies
cd frontend
npm install

# Backend dependencies (Gradle wrapper bootstrap)
cd ../backend
./gradlew --version
```

**Useful dev commands**
- `./gradlew bootRun` – start backend with local config.
- `VITE_API_BASE=http://localhost:8080/api npm run dev` (in `frontend/`) – start the Vite dev server (defaults to `5173`).
- `cd payment && npm install && npm start` – run the payment stub standalone on `http://localhost:9090`.
- `docker compose up -d --build` – full stack with Postgres, backend, payment stub, and compiled frontend on `http://localhost:3000`.

Environment configuration (backend):
| Variable | Default | Purpose |
|----------|---------|---------|
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/tickets` | Postgres connection string |
| `SPRING_DATASOURCE_USERNAME` | `tickets` | DB user |
| `SPRING_DATASOURCE_PASSWORD` | `tickets` | DB password |
| `AUTH_TOKEN_SECRET` | `local-secret` | Symmetric secret for auth token generation/validation |
| `PAYMENT_BASE_URL` | `http://localhost:9090` | Payment stub endpoint |

---

## Backend

Run locally:
```bash
cd backend
./gradlew bootRun
```

Build artifacts:
```bash
./gradlew clean build   # compiles + runs tests
```

Project structure follows controller → service → repository responsibilities. DTOs map external payloads; models encapsulate JPA entities (no column/index noise). Utilities and configuration live in the `util` and `config` packages.

Testing & coverage:
```bash
./gradlew test jacocoTestReport
```
- Unit & controller/service tests live under `src/test/java`.
- JaCoCo HTML report: `backend/build/reports/jacoco/test/html/index.html`.

Authentication flow:
- `POST /api/auth/mock` accepts `{ email, displayName, roles[], managedVenueIds[] }`.
- `GET /api/me` returns user profile, roles, and managed venues.

---

## Database Migrations & Seeding

Flyway is configured with:
- **Versioned schema migrations** under `backend/src/main/resources/db/migration/` (`V1__create_tables.sql`).
- **Repeatable data migrations** under `backend/src/main/resources/db/migration/repeatable/` (e.g., `R__seed_roles.sql`, `R__seed_sample_data.sql`).

Repeatable migrations run on every deployment, ensuring seeds (roles, admin user, venues, events) stay up to date without bumping schema versions. To add new reference data:
1. Create or edit a repeatable migration (`R__*.sql`) in the `repeatable/` directory.
2. Run `./gradlew flywayMigrate` (or restart the container/Compose stack) to apply updates.

For destructive testing or fresh starts:
```bash
docker compose down -v
docker compose up -d --build
```
This resets the Postgres volume and replays schema + seed migrations.

---

## Frontend

Run locally:
```bash
cd frontend
VITE_API_BASE=http://localhost:8080/api npm run dev   # serves on http://localhost:5173
```
You can store the base URL in `.env.local` (e.g., `VITE_API_BASE=http://localhost:8080/api`) to avoid exporting it each time.

Build & lint:
```bash
npm run build
npm run lint
```

Vitest unit/component tests:
```bash
npm run test              # unit/component tests
npm run test:coverage     # generate HTML + LCOV coverage (V8)
```
Coverage output: `frontend/coverage/index.html` when using `test:coverage`.

Mock SSO login tips:
- Email & display name are required.
- Check "Sign in with manager role" to unlock manager UI. Provide managed venue IDs (comma-separated UUIDs) to grant venue access (sample IDs seeded via Flyway).
- Admin role exposes aggregated venue metrics.
- After a successful login you are redirected to `/`; the role switcher navigates between `/` (attendee), `/manager`, and `/admin`.

---

## Testing & Coverage

| Scope | Command | Output |
|-------|---------|--------|
| Backend unit/controller/service tests | `./gradlew test` | JUnit reports in `backend/build/reports/tests/test` |
| Backend coverage | `./gradlew jacocoTestReport` | HTML in `backend/build/reports/jacoco/test/html/index.html`, LCOV available |
| Frontend unit/component tests | `npm run test` | Fast dev run (no coverage by default) |
| Frontend coverage | `npm run test:coverage` | HTML/LCOV under `frontend/coverage/` |
| End-to-end tests (Playwright) | See [End-to-End Testing](#end-to-end-testing) | HTML report under `frontend/coverage/playwright-report/` |
| Frontend lint | `npm run lint` | ESLint (React, hooks, refresh) |

---

## End-to-End Testing

Playwright specs live in `frontend/tests/e2e/` and target the compiled frontend served by nginx.

1. Ensure the stack is running (compile + services):
   ```bash
   docker compose up -d --build
   ```

2. Run Playwright (Chromium project by default):
   ```bash
   cd frontend
   PLAYWRIGHT_BASE_URL=http://localhost:3000 npm run test:e2e
   ```
   - Reports: `frontend/coverage/playwright-report/index.html`.
   - To open the latest report:
     ```bash
     npx playwright show-report coverage/playwright-report
     ```

Tests cover attendee, manager, and admin flows, ensuring mock authentication, role switching, and venue/event visibility behave correctly.

---

## Docker Orchestration

`docker-compose.yml` provisions:
- `postgres` – PostgreSQL 16 with a dedicated volume.
- `payment` – Node/Express payment stub (port `9090`).
- `backend` – Spring Boot app (port `8080`), built with Temurin 25.
- `nginx` – Serves the built frontend at `http://localhost:3000`, proxying `/api` to the backend.

Run the stack:
```bash
docker compose up -d --build
```

Stop & clean (removes DB volume too):
```bash
docker compose down -v
```

Logs:
```bash
docker compose logs -f backend
docker compose logs -f payment
docker compose logs -f nginx
```

---

## Simulating a Fresh Deployment

To mimic an environment from scratch (useful before releases or when validating migrations):
1. `docker compose down -v` – tear down services and remove volumes.
2. `./gradlew clean` (backend) and delete `frontend/dist`/`node_modules` if you want a pristine build.
3. `docker builder prune` *(optional)* – clear cached layers so Docker rebuilds everything.
4. `docker compose up -d --build` – rebuild images, replay schema + repeatable seeds.
5. Run the full test suite:
   ```bash
   ./gradlew test
   npm run test
   npm run test:coverage
   PLAYWRIGHT_BASE_URL=http://localhost:3000 npm run test:e2e
   ```

Following this process surfaces migration issues, dependency drift, and integration regressions before shipping.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Backend fails with Java toolchain error during Docker build | Host lacks Java 25 | Use the provided Docker build (Temurin 25) or install JDK 25 locally |
| `Unable to ignore case of ... Role` error during mock login | Stale schema/data | Reset DB: `docker compose down -v && docker compose up -d --build` |
| Playwright tests fail with network/401 errors | Stack not running or wrong base URL | Start Compose stack and set `PLAYWRIGHT_BASE_URL=http://localhost:3000` |
| Manager view missing venues | Mock login lacks `managedVenueIds` | Supply seeded UUIDs (see repeatable migration) when checking manager role |
| Payment calls fail | Payment stub not reachable | Ensure `payment` service is up (port `9090`), adjust `PAYMENT_BASE_URL` if running outside Compose |

---

## Additional Resources

- `backend/HELP.md` – Spring Boot generated starter tips.
- Playwright docs: <https://playwright.dev>
- Flyway repeatable migrations: <https://documentation.red-gate.com/fd/repeatable-migrations-184127470.html>
