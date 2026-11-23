# ADR 0001 — Use Yjs and Hocuspocus for real-time collaboration

- **Status:** Accepted
- **Date:** 2026-11-22
- **Deciders:** Project maintainer
- **Related:** I-07, I-08, I-09 (see `architecture/invariants.md`)

---

## Context

Wanderers-Quill must let two or more people edit the same world timeline in real
time. The core loop of v0.1 is: *a signed-in user creates a timeline event, and
another signed-in user sees it appear in real time.*

Real-time collaborative editing is a hard problem. Two users can edit the same
field at the same moment, or edit different fields of the same object, or one
user can be offline while the other makes changes. A naive implementation
("send the changed fields over a WebSocket and apply them on the other side")
breaks down as soon as edits conflict or arrive out of order.

We need a synchronization model that:

1. Resolves concurrent edits automatically, without a central lock.
2. Works with an intermittent connection (the collaborator may lose Wi-Fi).
3. Does not require us to write and maintain conflict-resolution logic ourselves.
4. Is open source and compatible with AGPL-3.0.
5. Runs on modest hardware (a Raspberry Pi).

## Decision

We will use **Yjs** as the data layer for all real-time state, and
**Hocuspocus** as the WebSocket synchronization server.

- **Yjs** is an open-source (MIT) implementation of CRDTs (Conflict-free
  Replicated Data Types). It merges concurrent edits automatically and
  deterministically, without a central authority. It is mature, widely used,
  and has bindings for the frameworks we use.
- **Hocuspocus** is an open-source (MIT) WebSocket server built on top of Yjs.
  It handles connection management, authentication hooks, and persistence
  hooks, so we do not have to write a WebSocket server from scratch.

All real-time state lives in Yjs documents. There is no custom diff/patch logic
anywhere in the codebase (see invariant I-07). Hocuspocus is the only WebSocket
server (see invariant I-08). The client never writes to the database directly
(see invariant I-09).

## Consequences

### Positive

- **Conflict resolution is solved.** Yjs merges concurrent edits deterministically.
  We do not write or maintain merge logic.
- **Offline tolerance comes for free.** CRDTs are designed for intermittent
  connectivity; a collaborator who loses connection can reconnect and sync.
- **Proven ecosystem.** Yjs is used by many production collaborative apps and
  has extensive documentation and community support.
- **AGPL-3.0 compatible.** Both Yjs and Hocuspocus are MIT-licensed, which is
  compatible with our AGPL-3.0 license.
- **Persistence is pluggable.** Hocuspocus exposes a persistence hook, so we can
  write the serialized Yjs state to PostgreSQL on every change without coupling
  the sync layer to the database.

### Negative

- **Learning curve.** CRDTs are conceptually different from traditional
  request/response state. Contributors (and AI agents) must understand that
  state is a replicated document, not a row in a table.
- **Debugging is less obvious.** When something goes wrong, the state is a
  binary Yjs document, not a readable JSON blob. We mitigate this by exposing a
  debug endpoint that serializes the Yjs state to JSON.
- **Hocuspocus is a dependency we do not control.** If its maintainers abandon
  it, we would need to fork it or replace it with a raw Yjs + `ws` server. This
  is acceptable: Hocuspocus is a thin layer over Yjs, and the Yjs document
  format is the real contract.
- **Payload size.** Yjs documents carry metadata for conflict resolution, so
  they are larger than plain JSON. At our scale (two to ten users, text-based
  timelines), this is negligible.

## Alternatives considered

| Alternative | Why rejected |
| :--- | :--- |
| **Custom WebSocket sync (send changed fields)** | Requires us to write and maintain conflict-resolution logic. Breaks under concurrent edits. Rejected. |
| **Operational Transformation (OT)** | Requires a central server that orders operations. More complex than CRDTs for our use case, and less tolerant of offline edits. Rejected. |
| **Automerge** | Another CRDT library. Comparable to Yjs, but Yjs has a larger ecosystem, better performance for our workload, and Hocuspocus as a ready-made server. Rejected in favor of Yjs. |
| **Firebase / Supabase Realtime** | Proprietary or hosted services. Violates the self-hosted, surveillance-free principle. Rejected. |
| **ShareDB** | OT-based, requires a central server, less offline-tolerant. Rejected. |

## References

- Yjs documentation: https://docs.yjs.dev/
- Hocuspocus documentation: https://tiptap.dev/hocuspocus
- Yjs GitHub repository: https://github.com/yjs/yjs
- Hocuspocus GitHub repository: https://github.com/ueberdosis/hocuspocus