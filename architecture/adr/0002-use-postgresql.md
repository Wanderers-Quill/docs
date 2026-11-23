# ADR 0002 — Use PostgreSQL as the database

- **Status:** Accepted
- **Date:** 2026-11-22
- **Deciders:** Project maintainer
- **Related:** I-01, I-02 (see `architecture/invariants.md`)

---

## Context

Wanderers-Quill stores world data: timeline events, and eventually characters,
locations, factions, and lore pages. The data model is expected to evolve as
the project grows and as users adapt it to their own worlds.

Two properties of the data matter for the database choice:

1. **Semi-structured attributes.** A timeline event is not just `title` and
   `date`. Users will want custom fields ("factions involved", "canon level",
   "cross-references") that we cannot anticipate at schema-design time. The
   database must store and query semi-structured data efficiently.
2. **A future geospatial module.** The roadmap includes a map module. Maps need
   geospatial queries (points, polygons, distances), and we want a database
   with a mature geospatial extension rather than a bolt-on.

We also need the database to run on modest hardware (a Raspberry Pi) and to be
deployable with Docker in a single command.

## Decision

We will use **PostgreSQL** as the database for Wanderers-Quill.

In the MVP, we use PostgreSQL with hand-written SQL (see invariant I-01). We do
not use an ORM. All queries live in `core/src/db/queries/`.

For semi-structured attributes, we use PostgreSQL's **JSONB** type, which
stores JSON in an optimized binary format and supports GIN indexing and rich
operators (`@>`, `?`, `jsonb_path_query`, etc.).

For the future map module, we plan to use **PostGIS**, the mature open-source
geospatial extension for PostgreSQL.

## Consequences

### Positive

- **JSONB gives us schema flexibility without giving up query power.** We can
  store custom event attributes as JSONB and index them with GIN, so queries
  like "all events that mention faction X" remain fast without a schema
  migration.
- **PostGIS is the standard for geospatial data.** When the map module arrives,
  we will not need to change databases or add a separate geospatial store.
- **Rich extension ecosystem.** `pgvector` (semantic search over lore),
  `pg_cron` (scheduled tasks), and `TimescaleDB` (time-series, useful for
  timelines) are all available if the project needs them.
- **Strong standards compliance and data integrity.** PostgreSQL enforces
  constraints, foreign keys, and transactions reliably, which matters for
  world data that users expect to be consistent.
- **BSD license.** PostgreSQL's license is permissive and compatible with our
  AGPL-3.0 license.
- **Docker deployment is one line.** The official `postgres` image runs on a
  Raspberry Pi with no tuning for our scale.

### Negative

- **More operational complexity than SQLite at large scale.** PostgreSQL
  requires tuning (`shared_buffers`, connections, etc.) at production scale.
  At our scale (two to ten users, a Raspberry Pi), this is irrelevant: the
  default Docker configuration is sufficient.
- **Not MySQL-compatible.** If a contributor comes from a MySQL/MariaDB
  background, some syntax differs (e.g., `SERIAL` vs `AUTO_INCREMENT`,
  `RETURNING` clauses). This is a minor learning cost, documented in
  `development/setup.md`.
- **Slightly heavier than SQLite.** PostgreSQL runs as a separate process,
  whereas SQLite is a single file. For a self-hoster, this means one more
  container in `docker-compose.yml`. Acceptable.

## Alternatives considered

| Alternative | Why rejected |
| :--- | :--- |
| **SQLite** | Excellent for a single writer, but weak for concurrent writes from multiple collaborators. Also lacks JSONB indexing and PostGIS. We may use SQLite for local development, but production uses PostgreSQL. |
| **MariaDB** | JSON support is partial (no `@>` operators, requires generated columns to index JSON). No PostGIS equivalent. Its main advantage is MySQL compatibility, which we do not need. Rejected. |
| **MongoDB** | Document store, good for semi-structured data, but weaker transactional guarantees and no mature geospatial extension comparable to PostGIS. Also, a document store would push us away from relational integrity for world data. Rejected. |
| **CockroachDB / YugabyteDB** | Distributed SQL databases. Overkill for a two-user self-hosted app, and heavier to run on a Raspberry Pi. Rejected. |

## References

- PostgreSQL documentation: https://www.postgresql.org/docs/
- JSONB documentation: https://www.postgresql.org/docs/current/datatype-json.html
- PostGIS documentation: https://postgis.net/documentation/
- pgvector: https://github.com/pgvector/pgvector