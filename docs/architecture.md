# ClubIQ Architecture

This document provides a comprehensive overview of the ClubIQ system architecture, detailing its components, technologies, and interactions.

---

## 1. System Overview

**ClubIQ** is a full-stack platform designed for club management, including member tracking, activity planning, task assignment, and performance rating. The system is built with a modular architecture, leveraging modern web technologies for scalability and ease of development.

At runtime in Docker Compose, the system is organized as four services connected on an internal bridge network:

- postgres (data store)
- backend (Flask API on Gunicorn)
- frontend (Next.js App Router application)
- nginx (single external entrypoint)

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
- **WSGI Server:** [Gunicorn](https://gunicorn.org/) (`wsgi:app`, configured in `gunicorn_config.py`)

### Frontend

- **Framework:** [Next.js](https://nextjs.org/) (App Router, React 19)
- **Styling:** [Tailwind CSS](https://tailwindcss.com/)
- **UI Components:** [Radix UI](https://www.radix-ui.com/) / [Shadcn UI](https://ui.shadcn.com/)
- **Authentication:** Clerk for Next.js
- **State Management:** React Context API (`AppContext`) & [TanStack Query](https://tanstack.com/query/latest)
- **Icons:** [Lucide React](https://lucide.dev/)

### Infrastructure & DevOps

- **Containerization:** [Docker](https://www.docker.com/) & [Docker Compose](https://docs.docker.com/compose/)
- **Reverse Proxy / Web Server:** [nginx](https://nginx.org/) (host port `80`, reverse proxy in front of backend and frontend)
- **Build Tool:** `Makefile`

---

## 3. Project Structure

### Root Directory

- `docker-compose.yml`: Defines the multi-container environment.
- `Makefile`: Provides shortcuts for common development tasks (build, up, down).
- `docs/`: System documentation (including this file).
- `Backend/`: The Flask REST API service.
- `Frontend/`: The Next.js frontend application.
- `nginx/`: Reverse proxy configuration.

### Backend Structure (`/Backend`)

- `app/`: Main application package.
  - `activities/`, `auth/`, `clubs/`, `invitations/`, `members/`, `rating/`: Modular blueprints/resources.
  - `health/`: Health check resource (`/api/backend-health`).
  - `models.py`: Database schema definitions using SQLAlchemy.
  - `__init__.py`: App factory and extension initialization.
- `migrations/`: Database migration scripts.
- `endpoint_documentation/`: Detailed documentation for each API module.
- `config.py`: Application configuration and environment variables.
- `wsgi.py`: Entry point for the production server.
- `gunicorn_config.py`: Gunicorn configuration settings.
- `entrypoint.sh`: Startup script for the Docker container.

### Frontend Structure (`/Frontend`)

- `src/app/`: Next.js App Router pages and layouts.
  - `admin/`: Admin-specific dashboards and management views.
  - `member/`: Member-specific views and tasks.
  - `auth/`: Authentication pages (Sign-in, Sign-up).
  - `frontend-health/route.tsx`: Frontend health route (`/frontend-health`).
- `src/components/`: Reusable React components.
  - `admin/`, `member/`, `reusables/`, `ui/`: Component categorization.
- `src/context/`: React Context providers (e.g., `AppContext`).
- `src/hooks/`: Custom React hooks.
- `src/DAL/`: Data Access Layer (e.g., Clerk role management).
- `src/utils/`: Helper functions and utilities.
- `src/styles/`: Global CSS and component-specific styles.

---

## 4. Database Schema & Models

The system uses a relational schema managed by PostgreSQL:

- **User:** Stores user profile data synced from Clerk.
- **Club:** Represents a club/organization.
- **ClubMember:** Junction table linking Users to Clubs with specific roles.
- **Activity:** Represents an event or project within a club.
- **Task:** Granular tasks within an Activity, assigned to specific Users.
- **Rating:** Performance evaluations for Users on specific Tasks.
- **Invitation:** Tracks invitations sent to potential members to join a club.

Notable model details from the current schema:

- Club, Activity, Task, Rating, and Invitation IDs are UUIDs.
- User IDs are integer primary keys.
- Task status enum values: `pending`, `in_progress`, `completed`.
- Invitation status enum values: `pending`, `in_progress`, `accepted`.

---

## 5. Authentication & Security

### Clerk Integration

The platform offloads identity management to [Clerk](https://clerk.com/).

- **Frontend:** Uses Clerk's Next.js SDK to handle sessions, sign-ins, and protected routes via `clerkMiddleware`.
- **Backend:** Validates Clerk-issued JWTs locally using **JWKS (JSON Web Key Sets)**. This "networkless" verification ensures high performance and security.

### Authorization Decorators

The backend implements a custom `@auth_required` decorator:

- Verifies the JWT signature.
- Maps the `clerk_id` (from the JWT `sub` claim) to the internal `User` ID.
- Enforces role-based access control (RBAC).

### User Synchronization

Since primary user data resides in Clerk, the system implements a **Sync Flow**:

1. User logs into the Frontend via Clerk.
2. Frontend calls `POST /api/auth/sync/` with the JWT.
3. Backend extracts user details from JWT claims and ensures a corresponding record exists in the local PostgreSQL database.

The backend also exposes `GET /api/auth/test/` for auth verification and `GET /api/auth/me/<user_id>/` for profile retrieval.

---

## 6. Deployment & Containerization

The system is fully containerized using Docker Compose, consisting of four primary services:

1. **postgres:** PostgreSQL 16 with named volume `postgres-data`.
2. **backend:** Flask container built from `Backend/Dockerfile`, started through `Backend/entrypoint.sh`, served by Gunicorn on port 5000.
3. **frontend:** Next.js container built from `Frontend/Dockerfile`, running `npm run dev` on port 3000.
4. **nginx:** Reverse proxy container (`nginx:alpine`) exposed on host port 80.

Only nginx is exposed to the host. Backend, frontend, and postgres are internal to `clubiq-network`.

### Startup Sequence

1. `docker-compose up` starts the containers.
2. The **backend** container's `entrypoint.sh` waits for the **postgres** container to be ready (using `pg_isready`).
3. `flask db upgrade` is executed automatically to ensure the schema is up to date.
4. The Flask app is launched using Gunicorn.
5. The **frontend** service starts once backend is healthy.
6. The **nginx** service starts after backend and frontend become healthy.

Current healthcheck endpoints:

- Backend: `/api/backend-health`
- Frontend: `/frontend-health`
- Nginx: `/nginx-health`

---

## 7. Data Flow

### API Request Lifecycle

1. Browser sends request to nginx (`http://localhost`).
2. Nginx routes `/api/*` to backend (`backend:5000`) and other paths to frontend (`frontend:3000`).
3. Frontend sends API requests (typically with Clerk JWT in `Authorization`).
4. Backend processes CORS, auth decorators, and resource handlers.
5. SQLAlchemy reads/writes PostgreSQL.
6. Backend returns JSON; frontend updates UI state.

---

## 8. Contributor Guidelines

- **Database Changes:** Always use `flask db migrate` and `flask db upgrade` to manage schema changes.
- **New Modules:** Follow the existing blueprint structure (`app/module_name/`).
- **Frontend Components:** Use the components in `src/components/ui/` (Shadcn) for consistent styling.
- **Security:** Never commit `backend.env` or `frontend.env` files. Use `backend.env.example` and `frontend.env.example` as templates for environment variables.

---

## 9. Operational Notes

- Backend and frontend source directories are bind-mounted in Docker Compose for iterative development.
- PostgreSQL data is persisted in the named volume `postgres-data`.
- Compose uses `depends_on` with `condition: service_healthy` to gate startup order.
- Root `.env` provides Compose interpolation for database credentials, and `Backend/backend.env` keeps the container runtime values aligned.
