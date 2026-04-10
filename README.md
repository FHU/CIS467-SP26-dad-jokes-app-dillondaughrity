# 🧔 Dad Jokes Central

A three-tier containerized web application for browsing, rating, and submitting dad jokes.

## Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Frontend   │─────▶│     API     │─────▶│  PostgreSQL  │
│  React/Vite  │      │   Express   │      │   Database   │
│  (nginx :80) │      │  (node :3000)│      │    (:5432)   │
└─────────────┘      └─────────────┘      └─────────────┘
     :8080                :3000                 :5432
```

| Layer    | Technology                  | Container Image     |
|----------|-----------------------------|---------------------|
| Frontend | React 18, Vite, TypeScript  | nginx:alpine        |
| API      | Express 4, TypeScript, pg   | node:20-alpine      |
| Database | PostgreSQL 16               | postgres:16-alpine  |

Both the frontend and API Dockerfiles use **multi-stage builds** to keep production images small.  The nginx container serves the compiled React app *and* reverse-proxies `/api/*` requests to the Express service so the browser only talks to a single origin.

## Prerequisites

- **Docker** (v20+) and **Docker Compose** (v2+)
- That's it — no local Node.js or PostgreSQL install required.

## Quick Start

```bash
# 1. Clone / download the project
cd dad-jokes-app

# 2. Build and start all containers
docker compose up --build

# 3. Open your browser
#    Frontend:  http://localhost:8080
#    API:       http://localhost:3000/api/health
```

The first run will:

1. Pull base images (`postgres:16-alpine`, `node:20-alpine`, `nginx:alpine`).
2. Build the API image (install deps → compile TypeScript → copy to slim runtime stage).
3. Build the frontend image (install deps → `vite build` → copy static files to nginx).
4. Start PostgreSQL, run `db/init.sql` to create the `jokes` table and seed 20 jokes.
5. Start the API (waits for the database health check to pass).
6. Start nginx to serve the frontend.

## Stopping & Cleaning Up

```bash
# Stop containers (preserves database data in the named volume)
docker compose down

# Stop AND delete the database volume (full reset)
docker compose down -v
```

## API Endpoints

| Method  | Endpoint              | Description                     |
|---------|-----------------------|---------------------------------|
| GET     | `/api/health`         | Health check (DB connectivity)  |
| GET     | `/api/jokes`          | List all jokes (?category=food) |
| GET     | `/api/jokes/random`   | Get one random joke             |
| GET     | `/api/jokes/:id`      | Get a specific joke             |
| POST    | `/api/jokes`          | Create a joke                   |
| PATCH   | `/api/jokes/:id/rate` | Rate a joke (0-5)               |
| DELETE  | `/api/jokes/:id`      | Delete a joke                   |
| GET     | `/api/categories`     | List distinct categories        |

### Example: Add a joke via curl

```bash
curl -X POST http://localhost:3000/api/jokes \
  -H "Content-Type: application/json" \
  -d '{"setup": "Why do Java developers wear glasses?", "punchline": "Because they can't C#!", "category": "tech"}'
```

## Project Structure

```
dad-jokes-app/
├── docker-compose.yml          # Orchestrates all 3 services
├── README.md
├── db/
│   └── init.sql                # Schema + seed data
├── api/
│   ├── Dockerfile              # Multi-stage: build TS → slim runtime
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       └── index.ts            # Express routes + pg connection
└── frontend/
    ├── Dockerfile              # Multi-stage: vite build → nginx
    ├── nginx.conf              # SPA routing + /api proxy
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    ├── index.html
    └── src/
        ├── main.tsx
        └── App.tsx             # React app (random, browse, add views)
```

## Key Docker / Compose Concepts Demonstrated

- **Multi-stage builds** — both app Dockerfiles compile in a `builder` stage and copy only production artifacts to the final image.
- **Named volumes** — `pgdata` persists database data across container restarts.
- **Bind-mount init scripts** — `db/init.sql` is mounted into the Postgres `docker-entrypoint-initdb.d` directory so it runs on first start.
- **Health checks** — the `db` service exposes a health check (`pg_isready`); the `api` service uses `depends_on: condition: service_healthy` to wait for it.
- **Service networking** — containers reference each other by service name (`db`, `api`) on the default Compose bridge network.
- **Reverse proxy** — nginx proxies `/api/*` to the `api` container, giving the browser a single origin and avoiding CORS issues.

## Local Development (Without Docker)

If you want to run the services directly on your machine:

```bash
# Terminal 1 — Start PostgreSQL (or use a local install)
docker compose up db

# Terminal 2 — API
cd api
npm install
export DB_HOST=localhost DB_USER=dadjokes DB_PASSWORD=dadjokes DB_NAME=dadjokes
npx ts-node src/index.ts

# Terminal 3 — Frontend (Vite dev server with hot reload)
cd frontend
npm install
npm run dev
# Note: Vite proxies /api to http://api:3000 by default.
# For local dev, change the proxy target in vite.config.ts to http://localhost:3000.
```

# Explanation Section

## Service Added: pgAdmin at localhost:5050

### Login:
  - Email: "admin@example.com"
  - Password: "admin" (implemented as a secret)

## Feature Added: 

### Joke of the Day:
  - Adds a tab on the navigation that will give you a different joke every day that won't change (until the date does).
