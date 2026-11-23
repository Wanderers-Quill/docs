# Development Setup

This guide gets a new contributor from zero to a running local instance of
Wanderers-Quill.

> **Scope:** This describes the development environment for the `core` and
> `timeline` repositories. For production deployment (Raspberry Pi, Cloudflare
> Tunnel), see `operations/backup-restore.md` and the deployment guide (coming
> in the Transition phase).

---

## Prerequisites

| Tool | Version | Why |
| :--- | :--- | :--- |
| **Node.js** | 20.9 or later | Required by Next.js  |
| **pnpm** | 9 or later | Workspace management for the monorepo |
| **Docker** | 24 or later | Runs PostgreSQL locally |
| **Docker Compose** | v2 or later | Orchestrates the local database |
| **Git** | Any recent version | Version control |

Install Node.js via your package manager or `nvm`. Install pnpm via
`corepack enable` (ships with Node.js 20+).

---

## 1. Clone the repositories

Wanderers-Quill is a multi-repository organization. For local development, you
need at least `core` and `timeline`:

```bash
mkdir wanderers-quill && cd wanderers-quill
git clone https://github.com/wanderers-quill/core.git
git clone https://github.com/wanderers-quill/timeline.git
git clone https://github.com/wanderers-quill/docs.git
```

Each repository has its own `package.json` and its own dependencies. The
`docs` repository has no code and needs no installation.

---

## 2. Start PostgreSQL

The local database runs in Docker. From the `core` repository:

```bash
cd core
cp .env.example .env
docker compose up -d postgres
```

The `docker-compose.yml` in `core` defines a PostgreSQL service with a named
volume for persistence and a healthcheck using `pg_isready` . The
database is ready when the healthcheck passes. To check:

```bash
docker compose ps
```

You should see `healthy` in the status column.

**Default credentials (development only):**

| Variable | Value |
| :--- | :--- |
| `POSTGRES_USER` | `wanderers` |
| `POSTGRES_PASSWORD` | `devpassword` |
| `POSTGRES_DB` | `wanderers_dev` |
| `POSTGRES_PORT` | `5432` |

These are defined in `core/docker-compose.yml` and `.env.example`. They are
**not** used in production. Never commit a real `.env` file.

---

## 3. Install dependencies

### Core (backend)

```bash
cd core
pnpm install
```

### Timeline (frontend)

```bash
cd ../timeline
pnpm install
```

Both repositories use pnpm workspaces internally. There is no shared root
workspace across repositories; each is independent, which matches the module
boundary defined in the architecture document (see invariant I-10).

---

## 4. Run database migrations

From `core`:

```bash
pnpm db:migrate
```

This applies the hand-written SQL migrations in `core/src/db/migrations/` to
your local PostgreSQL instance. See invariant I-01: there is no ORM, and
migrations are plain SQL files.

To seed the database with sample data (a demo world and timeline):

```bash
pnpm db:seed
```

---

## 5. Start the development servers

You need two terminals.

**Terminal 1 — Core (API + Hocuspocus):**

```bash
cd core
pnpm dev
```

This starts:
- The Next.js API server on `http://localhost:3001`
- The Hocuspocus WebSocket server on `ws://localhost:1234`

**Terminal 2 — Timeline (frontend):**

```bash
cd timeline
pnpm dev
```

This starts the Next.js frontend on `http://localhost:3000`.

Open `http://localhost:3000` in two browser tabs. Create an event in one tab;
it should appear in the other within two seconds. That is the core loop working
in development.

---

## 6. Run tests

### Core

```bash
cd core
pnpm test          # unit tests
pnpm test:integration  # integration tests (requires PostgreSQL running)
```

### Timeline

```bash
cd timeline
pnpm test
```

Per invariant I-17, no pull request merges with failing tests.

---

## 7. Environment variables reference

### Core (`.env`)

| Variable | Required | Default | Description |
| :--- | :---: | :--- | :--- |
| `DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `SESSION_SECRET` | Yes | — | Secret for session cookies (generate with `openssl rand -hex 32`) |
| `PORT` | No | `3001` | API server port |
| `HOCUSPOCUS_PORT` | No | `1234` | WebSocket server port |

### Timeline (`.env.local`)

| Variable | Required | Default | Description |
| :--- | :---: | :--- | :--- |
| `NEXT_PUBLIC_API_URL` | Yes | `http://localhost:3001` | Core API base URL |
| `NEXT_PUBLIC_WS_URL` | Yes | `ws://localhost:1234` | Hocuspocus WebSocket URL |

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
| :--- | :--- | :--- |
| `ECONNREFUSED` on port 5432 | PostgreSQL container not running | `docker compose up -d postgres` in `core` |
| Healthcheck stuck at `starting` | PostgreSQL still initializing | Wait 30–60 seconds; check `docker compose logs postgres` |
| WebSocket connection fails | Hocuspocus not running | Ensure `pnpm dev` in `core` is active |
| Changes not syncing between tabs | Two tabs on different worlds | Check the URL includes the same world ID |

---

## 9. What to read next

Before writing code, read:

1. [`architecture/architecture.md`](../architecture/architecture.md) — system overview
2. [`architecture/invariants.md`](../architecture/invariants.md) — hard rules
3. [`CONTRIBUTING.md`](https://github.com/wanderers-quill/.github/blob/main/CONTRIBUTING.md) — workflow