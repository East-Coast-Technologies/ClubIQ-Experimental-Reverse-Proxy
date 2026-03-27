# Club IQ Architecture

This document provides a comprehensive overview of the Club IQ system architecture, detailing its components, technologies, and interactions.

---

## 1. System Overview

**Club IQ** is a full-stack platform designed for club management, including member tracking, activity planning, task assignment, and performance rating. The system is built with a modular architecture, leveraging modern web technologies for scalability and ease of development.

All external traffic enters through an **Nginx reverse proxy**, which routes requests to the appropriate internal service (frontend or backend API).

---

## 2. Tech Stack

### Backend
- **Framework:** [Flask](https://flask.palletsprojects.com/) (Python 3.12)
- **API Style:** RESTful (using `flask-restful`)
- **Database:** [PostgreSQL](https://www.postgresql.org/)
- **ORM:** [SQLAlchemy](https://www.sqlalchemy.org/) with `flask-sqlalchemy`
- **Migrations:** `flask-migrate` (Alembic)
- **Authentication:** [Clerk](https://clerk.com/) (JWT-based)
- **Task Scheduling:** `flask-apscheduler`
- **Mail:** `flask-mail`
- **WSGI Server:** [Gunicorn](https://gunicorn.org/)

### Frontend
- **Runtime:** [Node.js](https://nodejs.org/) 20
- **Framework:** [Next.js](https://nextjs.org/) (App Router, React 19)
- **Bundler:** [Turbopack](https://turbo.build/pack) (configured via `next.config.js`)
- **Styling:** [Tailwind CSS](https://tailwindcss.com/) v4
- **UI Components:** [Radix UI](https://www.radix-ui.com/) / [Shadcn UI](https://ui.shadcn.com/)
- **Authentication:** Clerk for Next.js
- **State Management:** React Context API (`AppContext`) & [TanStack Query](https://tanstack.com/query/latest)
- **Forms:** [React Hook Form](https://react-hook-form.com/) with [Zod](https://zod.dev/) validation
- **Charts:** [Recharts](https://recharts.org/)
- **Notifications:** [Sonner](https://sonner.emilkowal.ski/)
- **Icons:** [Lucide React](https://lucide.dev/)

### Infrastructure & DevOps
- **Containerization:** [Docker](https://www.docker.com/) & [Docker Compose](https://docs.docker.com/compose/)
- **Reverse Proxy:** [Nginx](https://nginx.org/) (`nginx:alpine`) — single entry point for all traffic on port 80
- **Build Tool:** `Makefile`

---

## 3. Project Structure

### Root Directory
- `docker-compose.yml`: Defines the multi-container environment (postgres, backend, frontend, nginx).
- `Makefile`: Provides shortcuts for common development tasks (build, up, down, logs, shell access, migrations).
- `.env.example`: Template for root-level environment variables (PostgreSQL credentials).
- `Docs/`: System documentation (including this file).
- `Backend/`: The Flask REST API service.
- `Frontend/`: The Next.js frontend application.
- `nginx/`: Nginx reverse proxy configuration (`nginx.conf`).

### Backend Structure (`/Backend`)
- `app/`: Main application package.
  - `activities/`, `auth/`, `clubs/`, `invitations/`, `members/`, `rating/`: Modular blueprints/resources.
  - `health/`: Health check endpoint (`/health`) for container liveness probes.
  - `models.py`: Database schema definitions using SQLAlchemy.
  - `__init__.py`: App factory and extension initialization.
- `migrations/`: Database migration scripts managed by Alembic via `flask-migrate`.
- `tests/`: Automated test suite (pytest), with one test file per module.
- `endpoint_documentation/`: Detailed Markdown documentation for each API module.
- `config.py`: Application configuration (`Config` and `TestingConfig` classes).
- `run.py`: Flask application entry point (used by `FLASK_APP=run.py`); also provides `flask shell` context.
- `wsgi.py`: WSGI entry point consumed by Gunicorn in production.
- `gunicorn_config.py`: Gunicorn configuration settings (workers, threads, bind address, logging).
- `entrypoint.sh`: Docker container startup script — waits for PostgreSQL readiness, runs `flask db upgrade`, then launches Gunicorn.
- `backend.env.example`: Template for backend environment variables (database URL, Clerk keys, mail settings).

### Frontend Structure (`/Frontend`)
- `src/app/`: Next.js App Router pages and layouts.
  - `admin/`: Admin-specific dashboards and management views (activities, clubs, members, profile, ratings, reports).
  - `member/`: Member-specific views (activities, dashboard, profile, ratings, tasks, sign-up flow).
  - `auth/`: Authentication pages (Sign-in, Sign-up).
  - `health/`: Health check API route (`/api/health`) for container liveness probes.
- `src/components/`: Reusable React components.
  - `admin/`, `member/`, `reusables/`, `ui/`, `Loading/`: Component categorization.
- `src/context/`: React Context providers (`AppContext`).
- `src/hooks/`: Custom React hooks (`useSignOut`, `useToast`).
- `src/DAL/`: Data Access Layer — Clerk server actions for role management (`authServiceImpl.ts`).
- `src/lib/`: Shared utility functions (e.g., `utils.ts` for `clsx`/`tailwind-merge` helpers).
- `src/utils/`: Additional helper functions and utilities.
- `src/types/`: TypeScript type definitions (e.g., `task.tsx`).
- `src/mock/`: Mock/seed data used during development (e.g., `tasks.tsx`).
- `src/assets/`: Static assets (icons, images).
- `src/styles/`: Global CSS and component-specific styles.
- `src/middleware.ts`: Clerk middleware — enforces authentication on all non-public routes.
- `frontend.env.example`: Template for frontend environment variables (API URL, Clerk publishable key).

---

## 4. Database Schema & Models

The system uses a relational schema managed by PostgreSQL:

- **User:** Stores user profile data synced from Clerk (`clerk_id`, `name`, `email`, `username`, `role`).
- **Club:** Represents a club/organization.
- **ClubMember:** Junction table linking Users to Clubs with specific roles.
- **Activity:** Represents an event or project within a club, with optional start/end dates.
- **Task:** Granular tasks within an Activity, assigned to specific Users. Tracks status via `TaskStatusEnum` (`pending`, `in_progress`, `completed`).
- **Rating:** Performance evaluations for Users on specific Tasks (score + optional comments).
- **Invitation:** Tracks invitations sent to potential members to join a club. Tracks status via `InvitationStatusEnum` (`pending`, `in_progress`, `accepted`).

Primary keys use `UUID` (via `sqlalchemy.dialects.postgresql.UUID`) for Clubs, ClubMembers, Activities, Tasks, Ratings, and Invitations. User primary keys use auto-incrementing integers.

---

## 5. Authentication & Security

### Clerk Integration
The platform offloads identity management to [Clerk](https://clerk.com/).
- **Frontend:** Uses Clerk's Next.js SDK to handle sessions, sign-ins, and protected routes via `clerkMiddleware` (defined in `src/middleware.ts`). Public routes (`/`, `/auth/sign-in`, `/auth/sign-up`, `/api/webhooks/clerk`) are explicitly exempted.
- **Backend:** Validates Clerk-issued JWTs locally using **JWKS (JSON Web Key Sets)**. This "networkless" verification ensures high performance and security.

### Authorization Decorators
The backend implements a custom `@auth_required` decorator:
- Verifies the JWT signature.
- Maps the `clerk_id` (from the JWT `sub` claim) to the internal `User` ID.
- Enforces role-based access control (RBAC).

### Role Management
The frontend `src/DAL/authServiceImpl.ts` provides server actions (`setRole`, `removeRole`) that call the Clerk backend SDK to update a user's `publicMetadata.role`.

### User Synchronization
Since primary user data resides in Clerk, the system implements a **Sync Flow**:
1. User logs into the Frontend via Clerk.
2. Frontend calls `POST /api/auth/sync/` with the JWT.
3. Backend extracts user details from JWT claims and ensures a corresponding record exists in the local PostgreSQL database.

---

## 6. Deployment & Containerization

The system is fully containerized using Docker Compose, consisting of four services:

1. **postgres:** The PostgreSQL 16 relational database. Not exposed externally; accessible only on the internal `clubiq-network`.
2. **backend:** The Flask application, started via `entrypoint.sh` which runs Gunicorn. Depends on `postgres` being healthy.
3. **frontend:** The Next.js application, running in development mode (`npm run dev`). Depends on `backend` being healthy.
4. **nginx:** The Nginx reverse proxy (`nginx:alpine`). The only service with an externally exposed port (`80:80`). Routes traffic to `frontend` or `backend` based on path. Depends on all other services being healthy.

### Nginx Routing (`nginx/nginx.conf`)
- `GET /` and all non-API paths → proxied to `frontend:3000` (with WebSocket upgrade support for Next.js hot-reload).
- `GET /api/` and all API paths → proxied to `backend:8000`.
- `GET /health` → handled directly by Nginx (returns `200 "Nginx is healthy"`).

### Docker Network
All services communicate over a dedicated internal bridge network named `clubiq-network`. Only the `nginx` service exposes a port to the host.

### Environment Variables
Three separate `.env` files are used (not committed; use the provided `.example` templates):
- `.env` (root): PostgreSQL credentials (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`).
- `Backend/backend.env`: Flask/Gunicorn settings, database URL, Clerk keys, mail configuration.
- `Frontend/frontend.env`: Next.js public API URL and Clerk publishable key.

### Startup Sequence
1. `docker compose up` (or `make build`) starts all containers.
2. The **postgres** container starts and passes its health check (`pg_isready`).
3. The **backend** container's `entrypoint.sh` confirms PostgreSQL is ready, runs `flask db upgrade`, then launches Gunicorn.
4. The **frontend** container starts `npm run dev` once the backend is healthy.
5. The **nginx** container starts once all other services are healthy, and begins accepting traffic on port 80.

---

## 7. Data Flow

### API Request Lifecycle
1. **Client (Browser):** Sends an HTTP request to `http://localhost` (port 80).
2. **Nginx:** Inspects the request path.
   - Paths starting with `/api/` are forwarded to the backend.
   - All other paths are forwarded to the Next.js frontend.
3. **Frontend (Next.js):** For API calls, React components attach a Clerk JWT in the `Authorization: Bearer <token>` header before forwarding to the backend via Nginx.
4. **Backend (Middleware):** CORS headers are processed.
5. **Backend (Decorator):** `@auth_required` validates the JWT against Clerk's JWKS.
6. **Backend (Controller):** The resource logic executes, interacting with the database via SQLAlchemy.
7. **Backend (Response):** Returns JSON data through Nginx back to the frontend.
8. **Frontend:** Updates UI state (via TanStack Query or local state) to reflect the changes.

---

## 8. Testing

The backend includes an automated test suite located in `Backend/tests/`, built with [pytest](https://pytest.org/) and `pytest-flask`. There is one test file per API module:

- `test_activities.py`, `test_auth.py`, `test_clubs.py`, `test_invitations.py`, `test_members.py`, `test_rating.py`

Tests use the `TestingConfig` class (in `config.py`), which substitutes an in-memory SQLite database so no live PostgreSQL instance is required.

Run the test suite inside the backend container:
```bash
docker compose exec backend pytest
```
Or using the Makefile migration shortcut: `make migrate`.

---

## 9. Contributor Guidelines

- **Database Changes:** Always use `flask db migrate` and `flask db upgrade` to manage schema changes. The `make migrate` target automates this inside the running backend container.
- **New Backend Modules:** Follow the existing blueprint structure (`app/module_name/`). Register new blueprints in `app/__init__.py`.
- **Frontend Components:** Use the components in `src/components/ui/` (Shadcn) for consistent styling.
- **Security:** Never commit `.env` files. Use the corresponding `.example` files as templates:
  - `.env.example` → `.env` (root, PostgreSQL credentials)
  - `Backend/backend.env.example` → `Backend/backend.env`
  - `Frontend/frontend.env.example` → `Frontend/frontend.env`
- **Makefile:** Use `make help` to see all available development shortcuts (build, logs, shell access, service control).
