# Changelog

All notable changes to this project are documented here. Each entry corresponds to a commit and is timestamped with the date it was pushed.

---

## [e3e3f47] – 2026-04-05

### docs: clarify instructions for configuring frontend API URL in manual setup

- Improved wording in `README.md` for the manual (non-Docker) setup section: clarified how to configure the frontend to point to the backend when running both services without nginx.

**Files changed:** `README.md`

---

## [3e17eed] – 2026-04-05

### fix: correct backend and frontend health check endpoint paths

- Fixed leading-slash omission in `Backend/app/health/routes.py` route registrations (`backend-health` → `/backend-health`).
- Renamed `Frontend/src/app/frontend-health/route.tsx` → `route.ts` (no JSX used in the handler).
- Added `/frontend-health` to the Clerk middleware public-routes matcher in `Frontend/src/middleware.ts` so the health endpoint is accessible without authentication.

**Files changed:** `Backend/app/health/routes.py`, `Frontend/src/app/frontend-health/route.ts`, `Frontend/src/middleware.ts`

---

## [e6bfeb0] – 2026-04-05

### feat: rename health check endpoints and migrate nginx probe to `/nginx-health`

- Renamed backend health route from `/api/health` to `/api/backend-health` (and `/backend-health`) in `Backend/app/health/routes.py`.
- Moved frontend health route from `Frontend/src/app/health/` to `Frontend/src/app/frontend-health/` to avoid colliding with nginx's probe path.
- Updated all three `docker-compose.yml` healthcheck probes: backend now targets `/api/backend-health`, frontend targets `/frontend-health`, and nginx targets `/nginx-health`.
- Renamed the nginx health location block from `/health` to `/nginx-health` in `nginx/nginx.conf` and added `proxy_http_version 1.1` to the `/api/` proxy location.

**Files changed:** `Backend/app/health/routes.py`, `Frontend/src/app/frontend-health/route.tsx`, `docker-compose.yml`, `nginx/nginx.conf`

---

## [fdfcf68] – 2026-04-04

### chore: remove root `.env.example` and wire postgres service to `backend.env`

- Deleted the stale root `.env.example` file (superseded by `Backend/backend.env.example`).
- Added `env_file: ./Backend/backend.env` to the `postgres` service in `docker-compose.yml` so `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` are sourced from the same file as the backend, removing the need for a separate root `.env`.

**Files changed:** `.env.example`, `docker-compose.yml`

---

## [4921a5a] – 2026-04-04

### fix: add `-d "$POSTGRES_DB"` to `pg_isready` in entrypoint

- Updated `Backend/entrypoint.sh` to pass `-d "$POSTGRES_DB"` to `pg_isready`, ensuring the readiness check targets the actual application database rather than defaulting to a database matching the username.

**Files changed:** `Backend/entrypoint.sh`

---

## [2d8ccd1] – 2026-04-04

### docs: standardize product name to "ClubIQ" in architecture documentation

- Replaced all occurrences of "Club IQ" (with a space) with "ClubIQ" in `docs/architecture.md` title, introduction, and system overview for consistent branding.

**Files changed:** `docs/architecture.md`

---

## [6c95485] – 2026-04-03

### fix: update environment file references in docker-compose and nginx upstream usage

- Updated `docker-compose.yml` `env_file` entries for backend and frontend to point at the real per-developer env files (`backend.env` / `frontend.env`) instead of the `.example` templates.
- Changed `/api/` proxy pass in `nginx/nginx.conf` from `http://backend:5000` to `http://backend` (using the declared upstream) to avoid configuration drift.
- Minor security-note improvements in `docs/architecture.md`.

**Files changed:** `docker-compose.yml`, `nginx/nginx.conf`, `docs/architecture.md`

---

## [Unreleased] – 2026-04-05

### docs: sync Docker and architecture documentation with current runtime behavior

- Updated `docs/Docker.md` to match the active `docker-compose.yml` topology: four services (`postgres`, `backend`, `frontend`, `nginx`) with only nginx exposed on host port `80`.
- Corrected healthcheck documentation to reflect current endpoints: backend `/api/backend-health`, frontend `/frontend-health`, and nginx `/nginx-health`.
- Updated environment setup instructions to use `Backend/backend.env.example` and `Frontend/frontend.env.example`, and documented the required root `.env` Compose interpolation values for `POSTGRES_*`.
- Updated access instructions so users open the app via `http://localhost` (nginx) rather than direct backend/frontend host ports.
- Updated `docs/architecture.md` to match current startup behavior (`entrypoint.sh` -> `pg_isready` wait -> `flask db upgrade` -> Gunicorn), network exposure, routing flow through nginx, and current model/service details.
- Performed a second-pass onboarding sync in `README.md` so top-level setup, access URLs, and Docker/nginx flow now match the updated documentation.
- Added a root `.env.example` template and restored documentation for root `.env` Compose interpolation alongside `Backend/backend.env`.

**Files changed:** `.env.example`, `README.md`, `docs/Docker.md`, `docs/architecture.md`, `docs/CHANGELOG.md`

---

## [0bfe4f0] – 2026-04-01

### revert: restore `wget --spider` in healthchecks per user testing confirmation

- Reverted the frontend and nginx Docker Compose healthcheck probes from `wget -qO-` back to `wget --no-verbose --spider`, which was confirmed to work correctly on both `node:20-alpine` and `nginx:alpine` BusyBox `wget` implementations.

**Files changed:** `docker-compose.yml`

---

## [3caf9a0] – 2026-04-01

### fix: address review comments – healthchecks, env files, entrypoint command, and docs

- Removed the `command: flask run ...` override from the backend service in `docker-compose.yml` so the entrypoint's Gunicorn invocation is not shadowed.
- Updated the backend and frontend `env_file` references to point to `backend.env` and `frontend.env` (copied from their respective `.example` files by contributors during setup).
- Updated `docs/architecture.md` to reflect that pgAdmin has been removed and nginx is the reverse proxy, and corrected the startup sequence to describe the `pg_isready`-based readiness check.

**Files changed:** `docker-compose.yml`, `docs/architecture.md`

---

## [e1f6c30] – 2026-04-01

### fix: update `.gitignore` and Docker documentation, correct service names and environment file references

- Added `backend.env` and `frontend.env` to `.gitignore` so secrets are not accidentally committed.
- Updated `docs/Docker.md` to use the correct service names and new environment file names (`backend.env.example` / `frontend.env.example`).
- Corrected the nginx `upstream backend` reference in `nginx/nginx.conf` to point to `backend:5000` (the correct port for the Flask/Gunicorn service).
- Updated `docs/architecture.md` to replace the pgAdmin reference with nginx.

**Files changed:** `.gitignore`, `docs/Docker.md`, `docs/architecture.md`, `nginx/nginx.conf`

---

## [78fa49a] – 2026-03-31

### fix: update environment files for Flask and Clerk settings, enhance Makefile and docker-compose health checks

- Added missing `FLASK_APP` and `FLASK_ENV` variables to `Backend/backend.env.example`.
- Added missing Clerk environment variable placeholders to `Frontend/frontend.env.example`.
- Corrected Makefile targets that referenced outdated variable or service names.
- Removed duplicate or conflicting entries from `docker-compose.yml`.

**Files changed:** `Backend/backend.env.example`, `Frontend/frontend.env.example`, `Makefile`, `docker-compose.yml`

---

## [19226aa] – 2026-03-31

### fix: update Dockerfile and environment files, improve healthcheck endpoints in docker-compose and nginx configuration

- Renamed `Backend/.env.example` → `Backend/backend.env.example` and `Frontend/.env.example` → `Frontend/frontend.env.example` for clearer distinction between services.
- Updated `Backend/Dockerfile` to use a relative path for the entrypoint script.
- Streamlined `docker-compose.yml`: corrected healthcheck URLs to use `/api/health` for backend and `/health` for frontend, and updated `env_file` references to the renamed example files.
- Fixed `nginx/nginx.conf` upstream backend port from `8000` to `5000`.

**Files changed:** `Backend/Dockerfile`, `Backend/backend.env.example` (renamed), `Frontend/frontend.env.example` (renamed), `docker-compose.yml`, `nginx/nginx.conf`

---

## [763e2de] – 2026-03-24

### fix: resolve nginx unhealthy service status and improve per-service healthcheck JSON responses

- Fixed the Docker Compose `depends_on` condition syntax: changed malformed YAML (empty `condition:` key) to the correct inline form `condition: service_healthy` for backend, frontend, and nginx services.
- Updated the backend health route (`Backend/app/health/routes.py`) to return a standardized JSON response.
- Updated the frontend health route (`Frontend/src/app/health/route.tsx`) to return a consistent JSON response.
- Added the `/health` location block to `nginx/nginx.conf` so nginx itself can respond to healthcheck probes.

**Files changed:** `Backend/app/health/routes.py`, `Frontend/src/app/health/route.tsx`, `docker-compose.yml`, `nginx/nginx.conf`

---

## [5fe5800] – 2026-03-23

### feat: current repo changes – nginx, healthchecks, Makefile overhaul, and Docker improvements

- Added the nginx service to `docker-compose.yml` with its own healthcheck and `depends_on` gating on backend and frontend.
- Added per-service healthchecks for postgres, backend, and frontend containers.
- Added a backend health API endpoint at `/api/health` (`Backend/app/health/routes.py`).
- Added a frontend health API route at `/health` (`Frontend/src/app/health/route.tsx`).
- Overhauled the `Makefile`: colorized output, improved `help` target, renamed `start-db` targets to `start-postgres`, removed pgAdmin targets, added nginx targets, and fixed `.PHONY` declarations.
- Removed the pgAdmin service from `docker-compose.yml`.
- Optimized Docker Compose build contexts for backend and frontend to reduce image build times.
- Updated `Backend/Dockerfile` paths and `Backend/entrypoint.sh` to use `pg_isready` for robust PostgreSQL readiness checks.
- Updated `Frontend/Dockerfile` for consistency.
- Updated the Next.js page title and metadata to "ClubIQ".
- Updated `Backend/.env.example` to include `POSTGRES_DB` and use it in `DATABASE_URL`.
- Expanded `.gitignore` with additional entries.

**Files changed:** `.gitignore`, `Backend/.env.example`, `Backend/Dockerfile`, `Backend/app/health/routes.py`, `Backend/entrypoint.sh`, `Frontend/Dockerfile`, `Frontend/src/app/health/route.tsx`, `Frontend/src/app/layout.tsx`, `Makefile`, `docker-compose.yml`, `nginx/nginx.conf`

---

## [37ae5ea] – 2026-03-13

### feat: add nginx reverse proxy and architecture documentation

- Added `nginx/nginx.conf` configuring nginx as a reverse proxy: routes `/api/` traffic to the Flask backend (port 5000) and all other traffic to the Next.js frontend (port 3000).
- Updated `docker-compose.yml` to include the nginx service and restructure the network topology so only nginx is exposed on host port 80; backend, frontend, and postgres communicate on the internal `clubiq-network`.
- Updated `Backend/Dockerfile` and `Frontend/Dockerfile` for improved build context alignment.
- Moved `Docker.md` into the `docs/` directory.
- Added `docs/architecture.md` documenting the full system architecture, tech stack, data models, API structure, and deployment topology.

**Files changed:** `Backend/Dockerfile`, `Frontend/Dockerfile`, `docker-compose.yml`, `docs/Docker.md` (moved from root), `docs/architecture.md` (new), `nginx/nginx.conf` (new)
