# Invariants — Wanderers-Quill

> **What this file is:** A list of rules that must hold true at all times in the
> codebase. If a change violates one of these, the change is wrong — even if the
> tests pass.
>
> **Who reads this:** Every contributor, human or AI, before touching code.
> If you are an AI agent, treat every rule here as a hard constraint.
>
> **How to use it:** If you are about to write code that breaks an invariant,
> stop. Either find a way to satisfy the invariant, or open an issue to discuss
> changing the invariant itself (which requires an ADR — see
> `architecture/adr/`).

---

## 1. Data & Persistence

### I-01 — No ORM. Hand-written SQL only.
All database access goes through the query layer in `core/src/db/queries/`.
No Prisma, Drizzle, TypeORM, Sequelize, or similar.  
**Why:** We support both SQLite (MVP) and PostgreSQL (planned). An ORM hides the
SQL dialect differences and makes the migration path harder to reason about.
Hand-written SQL keeps the queries explicit and portable.

### I-02 — The database is the source of truth for world data.
Yjs documents hold *live editing state* only. On every change, Hocuspocus
persists the serialized Yjs state to the database. The database is never
reconstructed from memory or from Yjs on startup.  
**Why:** The Yjs document is ephemeral; the database survives restarts.

### I-03 — All timestamps are stored in UTC.
No local time, no timezone offsets, no "date strings" in the database.  
**Why:** Two collaborators in different timezones must see the same event at the
same instant. Timezone conversion is a presentation concern, done in the frontend.

---

## 2. Authentication & Authorization

### I-04 — Auth lives entirely in `core`. No frontend module verifies tokens.
Frontend modules (`timeline`, future `map` and `wiki`) attach the session cookie
to requests and nothing else. They never decode, validate, or inspect the
session.  
**Why:** One auth implementation, one place to audit. A frontend module that
"also checks auth" is a second source of truth waiting to disagree with the first.

### I-05 — Every API endpoint under `/api/v1/` is authenticated by default.
Public endpoints (login, register, health check) are the explicit exception and
are listed in `core/src/api/public-routes.ts`. Adding a public route requires a
comment explaining why.  
**Why:** Default-deny. A forgotten `requireAuth()` is a security hole; a
forgotten "this is public" is a bug, not a breach.

### I-06 — Authorization is checked on the server, never trusted from the client.
The client may *hide* buttons it shouldn't show, but the server re-checks every
permission on every request.  
**Why:** The client is attacker-controlled. Hiding a button is UX; checking on
the server is security.

---

## 3. Real-Time Synchronization

### I-07 — All real-time state goes through Yjs. No custom diff/patch logic.
If two users need to see the same data change live, that data lives in a Yjs
document. There is no hand-rolled "send the changed fields over the socket"
logic anywhere.  
**Why:** Custom sync code is where collaboration bugs live. Yjs already solves
conflict resolution; re-solving it partially is worse than not solving it.

### I-08 — Hocuspocus is the only WebSocket server.
No second WebSocket server, no Socket.IO, no raw `ws` alongside it.  
**Why:** One sync authority. Two WebSocket servers means two connection states
that can disagree.

### I-09 — The client never writes to the database directly.
Frontend modules talk to `core` via HTTP or via Hocuspocus. Direct DB access from
the browser is impossible by design.  
**Why:** The database is not exposed to the network. Only `core` talks to it.

---

## 4. Module Boundaries

### I-10 — Frontend modules never import each other.
`timeline` does not import from `map`. `map` does not import from `wiki`.
Shared code lives in `core` (for backend) or in a future `shared-ui` package
(for frontend primitives).  
**Why:** Direct imports between modules turn a modular suite into a monolith with
extra steps. It also makes it impossible to deploy one module without the others.

### I-11 — All cross-module data flows through the `core` API.
If `timeline` needs a location that `map` created, it asks `core`, not `map`.  
**Why:** The `core` API is the contract. Modules that talk to each other directly
create N² contracts.

### I-12 — `core` does not know about specific frontend modules.
No `if (module === 'timeline')` anywhere in `core`. The core exposes generic
world-data APIs; modules decide what to do with them.  
**Why:** The core must outlive any single module. Coupling it to a module means
the core can never be reused or simplified.

---

## 5. Configuration & Environment

### I-13 — Configuration comes from environment variables only.
No config files committed to the repo, no hardcoded secrets, no `.env` files in
git. `.env.example` documents the variables; `.env` is gitignored.  
**Why:** One deployment mechanism (env vars) works for Docker, systemd, and
local dev. Config files drift; env vars are the lowest common denominator.

### I-14 — The app starts with zero configuration for the MVP.
`docker compose up` with no `.env` file must produce a working instance with
SQLite and a generated session secret.  
**Why:** A self-hoster should not need to read a manual to see the app run. Every
required env var is a reason someone gives up.

---

## 6. Licensing & Dependency Hygiene

### I-15 — The project is AGPL-3.0. All dependencies must be AGPL-compatible.
No proprietary dependencies. No dependencies with licenses incompatible with
AGPL-3.0 (e.g., SSPL, Commons Clause, non-commercial licenses).  
**Why:** One incompatible dependency can make the whole project undistributable.

### I-16 — Every new dependency needs a one-line justification in the PR.
"This adds `X` because `Y`." If the justification is "it's convenient", it is
not enough.  
**Why:** Dependencies are liabilities. A two-line PR that adds a 200kB dependency
should require an argument.

---

## 7. Testing & Quality Gates

### I-17 — No PR merges with failing tests.
CI must be green. No "merge now, fix later".  
**Why:** A red main branch means every contributor's next PR starts from a broken
base.

### I-18 — Every bug fix includes a regression test.
The test fails before the fix and passes after.  
**Why:** A fix without a test is a fix that can silently un-fix itself.

### I-19 — The architecture document is updated in the same PR that changes it.
If a PR changes a module boundary, adds an invariant, or makes an architectural
decision, `docs/architecture/architecture.md` (or a new ADR) is updated in that
same PR.  
**Why:** Documentation updated "later" is documentation that never gets updated.

---

## 8. What This File Is *Not*

- **It is not a style guide.** Formatting, naming, and lint rules live in
  `.editorconfig`, ESLint config, and `CONTRIBUTING.md`.
- **It is not a feature list.** Features live in the product vision document.
- **It is not permanent.** Invariants can change — but only through an ADR that
  explains why the old invariant no longer holds. The ADR is the record; this
  file is the current state.

---

## Quick reference (for AI agents)

If you are an AI agent working in this repo, check these before writing code:

| # | Rule | Where to look |
|---|------|---------------|
| I-01 | No ORM, hand-written SQL | `core/src/db/queries/` |
| I-04 | No auth logic in frontend | `timeline/src/` |
| I-05 | API is default-auth | `core/src/api/public-routes.ts` |
| I-07 | Real-time via Yjs only | `core/src/sync/` |
| I-10 | Modules don't import each other | `package.json` of each module |
| I-13 | Env vars only, no config files | `.env.example` |
| I-15 | AGPL-compatible deps only | `package.json`, `LICENSE` |