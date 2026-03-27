# 🐳 **ClubIQ Docker Guide**

## Overview

This document explains how to build and run the **ClubIQ** Club Management System locally using Docker.
It includes the **Flask backend**, **Next.js frontend**, **PostgreSQL database**, and an **Nginx reverse proxy**.

The setup automatically handles:

* Database initialization and migrations
* Hot reloading for both backend and frontend
* Persistent Postgres storage
* Unified entry point via Nginx (port 80)

---

## 1. Prerequisites

Before running the containers, ensure you have:

* **Docker** ≥ 20.x
* **Docker Compose** plugin ≥ v2.0
* **Make** (optional, for easier commands)

---

## 2. Project Services

| Service      | Description                                              | Internal Port | Exposed Port |
| ------------ | -------------------------------------------------------- | ------------- | ------------ |
| **nginx**    | Reverse proxy — routes `/api/` to backend, `/` to frontend | `80`        | `80`         |
| **frontend** | Next.js development server (Clerk auth integrated)       | `3000`        | —            |
| **backend**  | Flask API (Gunicorn, SQLAlchemy + migrations)            | `5000`        | —            |
| **postgres** | PostgreSQL 16 (persistent volume)                        | `5432`        | —            |

Only `nginx` is exposed to the host. All other services communicate over the internal `clubiq-network`.

---

## 3. Directory Layout

```bash
ClubIQ/
├── Backend/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── requirements.txt
│   ├── app/
│   │   ├── models.py
│   │   └── ...
│   └── backend.env.example
│
├── Frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   └── frontend.env.example
│
├── nginx/
│   └── nginx.conf
│
├── .env.example
├── docker-compose.yml
├── Makefile
├── .dockerignore
└── Docs/
    └── Docker.md
```

---

## 4. Environment Setup

Copy and configure the example environment files:

```bash
cp .env.example .env
cp Backend/backend.env.example Backend/backend.env
cp Frontend/frontend.env.example Frontend/frontend.env
```

Then open the three `.env` files and replace values as needed:

**ClubIQ/.env**

```bash
# Postgres Credentials
POSTGRES_USER=your-postgres-username
POSTGRES_PASSWORD=your-postgres-password
POSTGRES_DB=your-postgres-database
```

**Backend/backend.env**

```bash
# Flask settings
FLASK_APP=run.py
FLASK_ENV=development

# Database URL
DATABASE_URL=postgresql://<user>:<password>@postgres:5432/<db>

# Clerk Settings
CLERK_SECRET_KEY=your-clerk-secret-key
CLERK_ISSUER=https://...
CLERK_JWKS_URL=https://...
```

**Frontend/frontend.env**

```bash
# Clerk Settings
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=your-clerk-publishable-key
CLERK_SECRET_KEY=your-clerk-secret-key

# API URL (routed through Nginx)
NEXT_PUBLIC_API_URL=http://localhost
```

---

## 5. Build & Run

Run all containers with:

```bash
make build
```

This automatically:

* Builds all images
* Waits for PostgreSQL to be ready
* Runs `flask db upgrade` to apply any pending migrations
* Launches Gunicorn (backend), Next.js dev server (frontend), and Nginx (reverse proxy)

Visit your app at:

* **App (via Nginx):** [http://localhost](http://localhost)
* **Backend API (via Nginx):** [http://localhost/api/](http://localhost/api/)

To view the full list of commands provided through the Makefile, run:

```bash
make help
```

---

## 6. Manual Docker Commands

If you don't have `make` installed:

```bash
docker compose up --build
docker compose down
docker compose exec backend flask db upgrade
```

---

## 7. Possible Issues

| Problem                                          | Fix                                                                     |
| ------------------------------------------------ | ----------------------------------------------------------------------- |
| Containers build but backend crashes on startup  | Check `Backend/backend.env` and ensure `DATABASE_URL` uses `postgres` as the host |
| Frontend can't reach API                         | Confirm requests go through Nginx at `http://localhost/api/`            |
| Migrations not running                           | Run `make migrate` to apply migrations inside the backend container     |
| Database persists unwanted data                  | Run `make down-volumes` to remove containers and the Postgres volume    |
| Nginx health check fails                         | Ensure backend and frontend containers are healthy first                |

---
