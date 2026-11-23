# Wanderers-Quill — Architecture Overview

## 1. What problem does this system solve?

Wanderers-Quill is a self-hosted, collaborative worldbuilding suite.  
It lets small teams (2–10 people) build and edit story timelines, maps, and wikis together in real time, without surveillance, on their own hardware.

## 2. Codemap (top-level modules)

| Module | Repo | One-line responsibility |
|--------|------|-------------------------|
| **core** | `wanderers-quill/core` | Shared backend: auth, world data API, real-time sync server (Yjs + Hocuspocus) |
| **timeline** | `wanderers-quill/timeline` | Frontend module for chronological events |
| **map** | `wanderers-quill/map` | Frontend module for image-based maps (future) |
| **wiki** | `wanderers-quill/wiki` | Frontend module for lore pages (future) |

**Dependency direction:** `timeline` / `map` / `wiki` → `core`.  
No frontend module imports another frontend module directly.  
All cross-module data goes through the `core` API.

## 3. Invariants (rules that hold everywhere)

- **No ORM.** All database queries are hand-written SQL in `core/src/db/queries/`.
- **Auth is handled entirely by `core`.** Frontend modules never verify tokens; they only attach the session cookie.
- **Real-time state is managed by Yjs documents.** There is no custom diff/patch logic in the frontend.
- **No global state in frontend modules.** All state is local to the component or comes from the `core` API.
- **The `map` and `wiki` modules do not exist in the MVP.** The current codebase only implements `core` + `timeline`.

## 4. Boundaries

| Boundary | Rule |
|----------|------|
| `core` public API | Exposed via REST under `/api/v1/`. Frontend modules must use only these endpoints. |
| `core` internal | Everything under `core/src/internal/` is private. |
| Frontend modules | Each module is a separate deployable frontend. They communicate with `core` only via HTTP + WebSocket. |

## 5. Cross-cutting concerns

- **Auth:** Session cookie set by `core`, validated on every API request. No JWT in frontend.
- **Error handling:** `core` returns consistent JSON errors `{ error: { code, message } }`. Frontend modules display a generic fallback.
- **Logging:** Structured JSON logs from `core`. Frontend logs go to browser console only.
- **Configuration:** Environment variables only. No config files in the repo.
- **Database:** SQLite in MVP, PostgreSQL planned. Queries are written to be compatible with both where possible.

## 6. Architecture Decision Records (ADRs)

ADRs live in `docs/architecture/adr/`. Each file is named `NNNN-short-title.md`.  
See the folder for the current list.