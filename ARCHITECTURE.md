# Architecture

This document describes the high-level architecture, data flow, socket protocol, undo/redo model, performance decisions, and conflict resolution strategy for the Collaborative Canvas project.

## Overview

- Server: Express static server with Socket.IO (entry point: `server/server.js`)
- Client: Static files in `client/` (`index.html`, `canvas.js`, `main.js`, `websocket.js`)
- In-memory store: `server/rooms.js` maintains a Map of rooms. Each room contains:
  - `users`: Map of userId -> { socketId, userId, color }
  - `actions`: Array of action objects
- Action model: Each drawing operation is an action object with:
  - `id`: client-generated id
  - `points`: array of coordinates or batches of points
  - `tool`, `color`, `width`
  - `ts`: timestamp
  - `active`: boolean (true by default, false when undone)

## Data flow (textual)

User input (mouse/touch) -> client captures points and emits socket events:
- `stroke-start` (create action)
- `stroke-patch` (append points)
- `stroke-end` (finalize action)

Server receives events, updates the room's `actions` (merge or append), and broadcasts events back to the room. Clients receive events and update local action lists and canvas rendering.

Sequence example:
1. Client emits `stroke-start` with action id and initial points.
2. Client sends multiple `stroke-patch` events to stream additional points.
3. Client sends `stroke-end` to finalize the action.
4. Server merges or appends action to `room.actions` and broadcasts updates to other clients.
5. Clients apply server-sent actions and redraw.

## WebSocket protocol (Socket.IO events)

Client -> Server:
- `join` — `{ room, userId, color }`  
- `leave` — `{ room, userId }`  
- `stroke-start` — `{ room, action }`  
- `stroke-patch` — `{ room, id, points }`  
- `stroke-end` — `{ room, action }`  
- `stroke` — `{ room, action }` (full action)  
- `undo` — `{ room }`  
- `redo` — `{ room }`  
- `cursor` — `{ room, userId, x, y, tool }`

Server -> Client:
- `init` — `{ room, users, actions }` (sent to joining client)
- `users` — updated array of users
- `user-left` — `{ userId }`
- `stroke-start`, `stroke-patch`, `stroke-end`, `stroke` — action payloads
- `undo` — `{ id, action }`
- `redo` — `{ id, action }`
- `cursor` — `{ userId, x, y, room }`

Payload notes: actions should include `id`, `points`, `tool`, `width`, `color`, `ts`, and `active`.

## Undo/Redo model

- The server stores a chronological `room.actions` array.
- `undo`: server-side function `lastActiveAction(room)` finds the most recent action with `active === true`, sets `active = false`, and broadcasts `undo` with `{ id, action }`.
- `redo`: `lastUndoneAction(room)` finds the most recent `active === false` and sets `active = true` then broadcasts `redo`.
- This model is global per room; undo/redo changes are visible to all users in the room. The design favors a single authoritative server state for simplicity and consistency.

## Performance decisions and optimizations

- Offscreen canvas buffer: Clients render committed actions to an offscreen canvas and composite to reduce heavy redraws.
- Device pixel ratio handling: Clients scale canvases to support high-DPI displays.
- Action merging: Server `addAction` merges patches for the same action id to limit action count and reduce network noise.
- Committed ID tracking: Clients track `committedIds` to avoid drawing duplicates for actions that are finalized.
- Batching points in `stroke-patch`: Client batches points to limit the number of socket messages.

## Conflict resolution

- Actions are identified by client-generated ids. When a `stroke-patch` arrives, the server merges points into the existing action by id.
- Patches use a last-write-wins policy for overlapping fields. This is acceptable for typical single-author strokes, but may overwrite concurrent edits in rare cases.
- Server is the authoritative source for `room.actions`; clients reconcile local state with server broadcasts to avoid divergence.

## Suggested extensions and improvements

- Persistent storage: Use Redis or a database to persist `room.actions` so drawings survive restarts.
- Per-user undo stacks: Track per-user undo history and optionally translate local undos into global actions.
- CRDT-based merging: Implement CRDTs for stronger concurrent-edit guarantees for vector strokes.
- Authentication and rate limiting: Add auth and rate limiting to prevent abuse.
- Action pruning or snapshotting: Periodically compact action history or snapshot canvases to bound memory and redraw cost.

## Quick notes for maintainers

- Server entrypoint: `server/server.js`
- Client: `client/` (key files: `index.html`, `canvas.js`, `main.js`, `websocket.js`)
- In-memory store: `server/rooms.js`
- Action helpers: `server/drawing-state.js`
