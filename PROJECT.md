# spry

A monorepo with a FastAPI backend, a React frontend, and a Postgres database, all started by one command.

This document describes the **structure of the repository and the contracts between its parts**. It contains no implementation code.

---

## 1. Getting started

Prerequisite: **Docker Desktop**.

```
docker compose up
```

That is the only command a new developer runs. When the output settles:

| What | URL |
|---|---|
| Web app | http://localhost:5173 |
| API | http://localhost:8000/api/meetings |
| API docs (FastAPI built-in) | http://localhost:8000/docs |

No local Python, Node, or Postgres installation is needed or expected.

---

## 2. Repository layout

```
spry/
├── PROJECT.md
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── alembic.ini
│   ├── alembic/
│   │   ├── env.py
│   │   └── versions/
│   │       └── 0001_create_meetings.py
│   └── app/
│       ├── main.py
│       ├── config.py
│       ├── db.py
│       ├── models/
│       │   └── meeting.py
│       ├── schemas/
│       │   └── meeting.py
│       └── routers/
│           └── meetings.py
└── frontend/
    ├── Dockerfile
    ├── package.json
    ├── package-lock.json
    ├── index.html
    ├── vite.config.ts
    ├── tsconfig.json
    ├── tailwind.config.ts
    ├── postcss.config.js
    ├── components.json
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── index.css
        ├── types/
        │   └── meeting.ts
        ├── lib/
        │   ├── api.ts
        │   └── utils.ts
        └── components/
            ├── ui/
            ├── MeetingList.tsx
            └── MeetingForm.tsx
```

### 2.1 Root

| Path | Purpose |
|---|---|
| `PROJECT.md` | This document. The source of truth for structure and contracts. |
| `docker-compose.yml` | Defines the three services (`postgres`, `backend`, `frontend`), their ports, healthchecks, startup ordering, and the single named volume for database data. |
| `backend/` | The API service. Everything Python lives here. |
| `frontend/` | The web client. Everything Node lives here. |

### 2.2 `backend/`

Purpose: the HTTP API and everything that owns data. Built on **FastAPI**, with **SQLAlchemy** as the ORM and **Alembic** for schema migrations. Base image: `python:3.12-slim`.

| Path | Purpose |
|---|---|
| `Dockerfile` | Builds the backend image from `python:3.12-slim`. Its start command runs `alembic upgrade head` and then starts the ASGI server (see §5). |
| `requirements.txt` | Python dependencies, each pinned to an exact version. |
| `alembic.ini` | Alembic configuration. Contains no credentials; the database URL comes from the environment. |
| `alembic/` | Migration environment. |
| `alembic/env.py` | Connects Alembic to the SQLAlchemy metadata and reads the database URL from configuration. |
| `alembic/versions/` | One file per migration, in order. The first slice contains `0001_create_meetings.py`, which creates the `meetings` table (§4.3). |
| `app/` | The application package. |
| `app/main.py` | Creates the FastAPI application and mounts the routers. |
| `app/config.py` | Reads environment variables (§6) into a single settings object. The only place that reads the environment. |
| `app/db.py` | Creates the SQLAlchemy engine and session factory, and provides the per-request session dependency. |
| `app/models/` | SQLAlchemy table definitions. One file per table. |
| `app/schemas/` | Request and response shapes (Pydantic). These implement the JSON contract in §4. |
| `app/routers/` | One file per resource; each declares that resource's HTTP routes. |

Rule: routers depend on schemas, models, and `db.py`. Models never import schemas or routers.

### 2.3 `frontend/`

Purpose: the browser UI. Built on **React** with **Vite**, styled with **Tailwind**, using **shadcn/ui** components. Base image: `node:20-slim`.

| Path | Purpose |
|---|---|
| `Dockerfile` | Builds the frontend image from `node:20-slim`, installs dependencies from `package-lock.json`, and starts the Vite dev server (§5). |
| `package.json` / `package-lock.json` | Dependencies and scripts. Versions are pinned through the lockfile. |
| `index.html` | Vite entry HTML. |
| `vite.config.ts` | Dev server settings: listens on all interfaces on port 5173 and proxies `/api` to the backend (§4.1). |
| `tsconfig.json` | TypeScript configuration, including the `@/` path alias used by shadcn/ui. |
| `tailwind.config.ts`, `postcss.config.js` | Tailwind setup. |
| `components.json` | shadcn/ui configuration: where generated components go and which alias they use. |
| `src/main.tsx` | Mounts the React app. |
| `src/App.tsx` | The single page. Renders `MeetingForm` and `MeetingList` and coordinates refreshing the list after a successful create. There is no router. |
| `src/index.css` | Tailwind directives and theme variables. |
| `src/types/meeting.ts` | TypeScript types mirroring the JSON contract in §4. |
| `src/lib/api.ts` | The only module that calls `fetch`. Exposes "list meetings" and "create meeting". |
| `src/lib/utils.ts` | Small shared helpers required by shadcn/ui. |
| `src/components/ui/` | Components generated by the shadcn/ui CLI. Treated as vendored code. |
| `src/components/MeetingList.tsx` | Displays meetings returned by the API. |
| `src/components/MeetingForm.tsx` | Form for creating one meeting; shows validation errors returned by the API. |

---

## 3. Services (`docker-compose.yml`)

Three services, no others.

| | `postgres` | `backend` | `frontend` |
|---|---|---|---|
| Image / base | `postgres:16` | built from `backend/`, base `python:3.12-slim` | built from `frontend/`, base `node:20-slim` |
| Listens on | **5432** | **8000** | **5173** |
| Published to host | No (reachable only inside the compose network) | Yes, `8000:8000` | Yes, `5173:5173` |
| Depends on | nothing | `postgres` | `backend` |
| Dependency wait | n/a | `depends_on: postgres: condition: service_healthy` | `depends_on: backend: condition: service_healthy` |
| Own healthcheck | Yes | Yes | None (nothing depends on it) |
| Persistent storage | Named volume `pgdata` mounted at the Postgres data directory | none | none |
| Source mount for live reload | n/a | `./backend` mounted into the container | `./frontend` mounted into the container, with `node_modules` kept in the container (not shadowed by the host folder) |

### 3.1 Healthchecks

| Service | Check | Healthy means |
|---|---|---|
| `postgres` | `pg_isready` against the configured user and database | The server accepts connections. |
| `backend` | An HTTP `GET` to `http://localhost:8000/api/meetings`, issued with the Python standard library (the slim image has no `curl`) | The server is up, can reach the database, and the migrated `meetings` table exists. Healthy only on HTTP 200. |

Timing for both: `interval: 5s`, `timeout: 5s`, `retries: 10`. The backend adds `start_period: 15s` so that migrations running at boot are not counted as failures.

The backend deliberately reuses `GET /api/meetings` as its health probe rather than adding a dedicated health endpoint, because a successful response already proves the whole chain (process, database connection, schema).

### 3.2 Startup order

`docker compose up` brings the system up in this exact order:

1. **`postgres` starts.** Compose waits until its healthcheck reports healthy.
2. **`backend` starts** only once `postgres` is healthy. Its start command:
   1. runs `alembic upgrade head` (creates or updates the schema),
   2. then starts the API server on port 8000.
   Compose waits until the backend healthcheck reports healthy.
3. **`frontend` starts** only once `backend` is healthy. It starts the Vite dev server on port 5173.

The web app is usable after step 3. Because each step gates the next, the frontend never starts against an API that cannot answer, and the backend never starts against a database that cannot accept connections.

---

## 4. Contracts

### 4.1 Frontend ↔ backend transport

- The browser only ever talks to the frontend origin (`http://localhost:5173`). All API calls use **relative URLs beginning with `/api`**.
- The Vite dev server **proxies `/api/*` to the backend** at the address given by `API_PROXY_TARGET` (`http://backend:8000` in compose). Because of this proxy there is no CORS configuration anywhere.
- Request and response bodies are JSON with `Content-Type: application/json`.
- All routes are prefixed `/api`.

### 4.2 The Meeting resource

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Assigned by the server. Never sent on create. |
| `title` | string | 1–200 characters after trimming whitespace. |
| `starts_at` | string | ISO-8601 date-time with a timezone designator, e.g. `2026-10-05T09:00:00Z`. |
| `ends_at` | string | Same format as `starts_at`. Must be strictly later than `starts_at`. |
| `attendee_count` | integer | Zero or greater. |

Timestamp rules:
- **Responses** always emit UTC with a trailing `Z` (e.g. `2026-10-05T09:00:00Z`). No fractional seconds are guaranteed.
- **Requests** may use `Z` or an explicit offset (e.g. `+02:00`); the server converts to UTC. A timestamp **without** a timezone designator is rejected.
- The frontend converts between the user's local time and UTC; the backend never deals with local time.

### 4.3 Database table

Table `meetings` (created by migration `0001_create_meetings`):

| Column | Type | Constraints |
|---|---|---|
| `id` | integer | primary key, auto-increment |
| `title` | varchar(200) | not null |
| `starts_at` | timestamp with time zone | not null |
| `ends_at` | timestamp with time zone | not null |
| `attendee_count` | integer | not null, default 0, must be ≥ 0 |

Additional constraint: `ends_at` must be greater than `starts_at`.

The database column is named `attendee_count`, identical to the JSON field. No renaming happens between layers.

### 4.4 `GET /api/meetings`

Returns all meetings, ordered by `starts_at` ascending, then `id` ascending. No pagination or filtering in this slice.

Request: no body, no query parameters.

Response `200 OK`:

```json
{
  "items": [
    {
      "id": 1,
      "title": "Sprint planning",
      "starts_at": "2026-10-05T09:00:00Z",
      "ends_at": "2026-10-05T10:00:00Z",
      "attendee_count": 8
    }
  ]
}
```

With no meetings: `{"items": []}` (still `200`).

### 4.5 `POST /api/meetings`

Creates one meeting.

Request body (all four fields required; `id` must not be sent):

```json
{
  "title": "Sprint planning",
  "starts_at": "2026-10-05T09:00:00Z",
  "ends_at": "2026-10-05T10:00:00Z",
  "attendee_count": 8
}
```

Response `201 Created`: the created meeting, as a bare object (not wrapped in `items`):

```json
{
  "id": 1,
  "title": "Sprint planning",
  "starts_at": "2026-10-05T09:00:00Z",
  "ends_at": "2026-10-05T10:00:00Z",
  "attendee_count": 8
}
```

### 4.6 Errors

Validation failures return `422 Unprocessable Entity` in FastAPI's standard shape:

```json
{
  "detail": [
    {
      "loc": ["body", "ends_at"],
      "msg": "ends_at must be after starts_at",
      "type": "value_error"
    }
  ]
}
```

- `loc` is a list of strings/integers pointing at the offending field; the frontend uses `loc[-1]` to attach the message to a form field.
- `msg` is human-readable text; `type` is a machine-readable error code.
- Conditions that produce `422`: missing field, wrong type, empty or over-long `title`, negative `attendee_count`, timestamp without timezone, `ends_at` not after `starts_at`.
- Any unexpected server failure returns `500` with `{"detail": "Internal Server Error"}`.

---

## 5. How each service is started

| Service | Start behavior |
|---|---|
| `postgres` | Standard `postgres:16` entrypoint. On first start with an empty `pgdata` volume it creates the user and database given by its environment variables. |
| `backend` | Applies migrations (`alembic upgrade head`), then serves the app on `0.0.0.0:8000` with auto-reload on changes under the mounted `backend/` folder. |
| `frontend` | Serves the app with the Vite dev server on `0.0.0.0:5173` with hot module reload on changes under the mounted `frontend/` folder. |

---

## 6. Environment variables

All values are development-only and live in `docker-compose.yml`. There is no `.env` file.

| Service | Variable | Value | Meaning |
|---|---|---|---|
| `postgres` | `POSTGRES_USER` | `spry` | Database user to create. |
| `postgres` | `POSTGRES_PASSWORD` | `spry` | Password for that user. |
| `postgres` | `POSTGRES_DB` | `spry` | Database to create. |
| `backend` | `DATABASE_URL` | `postgresql+psycopg://spry:spry@postgres:5432/spry` | SQLAlchemy URL. Host `postgres` is the compose service name. Used by both the app and Alembic. |
| `frontend` | `API_PROXY_TARGET` | `http://backend:8000` | Where the Vite dev server forwards `/api`. |

---

## 7. First slice: scope

In scope:
- `GET /api/meetings` and `POST /api/meetings` (§4).
- One frontend page containing a meeting list and a form that creates a meeting. After a successful create, the list shows the new meeting without a manual page reload.
- One migration creating the `meetings` table.

Not in scope: update or delete endpoints, authentication, pagination, and any service beyond the three listed in §3 (no cache, queue, worker, reverse proxy, or orchestration manifests).