# Collaboration

SubTerm provides real-time collaborative editing, allowing multiple users to work on the same codebase simultaneously. This document explains how the collaboration system works.

## Overview

The collaboration system is built on [Yjs](https://yjs.dev/), a high-performance CRDT (Conflict-free Replicated Data Type) implementation. CRDTs enable real-time synchronization without requiring a central authority to resolve conflicts.

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│  Client A   │          │   Server    │          │  Client B   │
│  (Monaco)   │          │   (Yjs)     │          │  (Monaco)   │
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       │  join-file             │                        │
       │───────────────────────>│                        │
       │                        │                        │
       │  y-sync (full state)   │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │                        │        join-file       │
       │                        │<───────────────────────│
       │                        │                        │
       │                        │  y-sync (full state)   │
       │                        │───────────────────────>│
       │                        │                        │
       │  y-update (edit)       │                        │
       │───────────────────────>│                        │
       │                        │                        │
       │                        │       y-update         │
       │                        │───────────────────────>│
       │                        │                        │
       │                        │       y-update         │
       │<───────────────────────│<───────────────────────│
```

## How It Works

### 1. Document Initialization

When a user opens a file, the client:

1. Creates a new `Y.Doc` instance
2. Gets a shared text type (`doc.getText("monaco")`)
3. Connects to the collaboration socket
4. Joins the file's collaboration room

```javascript
// Client-side initialization
const doc = new Y.Doc();
const yText = doc.getText("monaco");

collabSocket.emit("join-file", { filePath, initialContent });
```

### 2. Server-Side Document Management

The server maintains a `Map<filePath, Y.Doc>` for all active documents:

```javascript
// Server-side document store
const docs = new Map();

socket.on("join-file", ({ filePath, initialContent }) => {
  let doc = docs.get(filePath);

  if (!doc) {
    doc = new Y.Doc();
    const yText = doc.getText("monaco");
    yText.insert(0, initialContent);
    docs.set(filePath, doc);
  }

  socket.join(filePath);
  socket.emit("y-sync", Y.encodeStateAsUpdate(doc));
});
```

### 3. Edit Propagation

When a user makes an edit:

1. Monaco Editor triggers an event
2. `MonacoBinding` converts it to a Yjs operation
3. Yjs generates a binary update
4. Update is sent via Socket.IO
5. Server broadcasts to all other clients
6. Remote clients apply the update

```javascript
// Client: Send update
doc.on("update", (update, origin) => {
  if (origin !== "remote") {
    collabSocket.emit("y-update", { filePath, update });
  }
});

// Server: Broadcast update
socket.on("y-update", ({ filePath, update }) => {
  const doc = docs.get(filePath);
  Y.applyUpdate(doc, update);
  socket.to(filePath).emit("y-update", { update });
});
```

### 4. Monaco Binding

The `y-monaco` package binds Yjs to Monaco Editor:

```javascript
import { MonacoBinding } from "y-monaco";

const binding = new MonacoBinding(
  yText, // Y.Text instance
  editor.getModel(), // Monaco model
  new Set([editor]), // Editor instances
  awareness, // Optional awareness
);
```

## Session Sharing

### Inviting Collaborators

Users can share their session by:

1. Clicking the "Invite" button
2. Copying the session URL
3. Sending it to collaborators

The URL format is:

```
https://subterm.example.com/webide?session=<sessionId>
```

### Joining a Session

When a collaborator opens a shared URL:

1. Client extracts `sessionId` from URL
2. Connects to the existing session's container
3. Opens the file tree (shared workspace)
4. Joins collaboration rooms for open files

```javascript
// Extract session from URL
const params = new URLSearchParams(window.location.search);
const sessionId = params.get("session");

// Connect to existing session
socket = io(`/workspace/${sessionId}`);
```

## Awareness Protocol

The awareness protocol tracks:

- Which users are connected
- Cursor positions
- Selection ranges
- User metadata (name, color)

```javascript
import * as awarenessProtocol from "y-protocols/awareness";

const awareness = new awarenessProtocol.Awareness(doc);

// Set local state
awareness.setLocalState({
  user: {
    name: "Alice",
    color: "#ff0000",
  },
  cursor: { line: 10, column: 5 },
});

// Listen for remote changes
awareness.on("change", ({ added, updated, removed }) => {
  // Update cursor widgets in editor
});
```

## Socket.IO Events

### Client → Server

| Event              | Payload                        | Description           |
| ------------------ | ------------------------------ | --------------------- |
| `join-file`        | `{ filePath, initialContent }` | Join document room    |
| `leave-file`       | `{ filePath }`                 | Leave document room   |
| `y-update`         | `{ filePath, update }`         | Send Yjs update       |
| `awareness-update` | `{ filePath, update }`         | Send awareness update |

### Server → Client

| Event              | Payload               | Description             |
| ------------------ | --------------------- | ----------------------- |
| `y-sync`           | `{ update }`          | Full document state     |
| `y-update`         | `{ update }`          | Incremental update      |
| `awareness-update` | `{ update }`          | Awareness state         |
| `collab-count`     | `{ filePath, count }` | Number of collaborators |
| `file-saved`       | `{ filePath }`        | File saved notification |

## Conflict Resolution

Yjs CRDTs automatically resolve conflicts without manual intervention:

### Example: Concurrent Edits

```
Initial: "Hello World"

Alice types "!" at end:  "Hello World!"
Bob types "?" at end:    "Hello World?"

After sync, both see:    "Hello World!?" or "Hello World?!"
(Deterministic order based on client IDs)
```

### CRDT Properties

| Property        | Description                           |
| --------------- | ------------------------------------- |
| **Commutative** | Order of operations doesn't matter    |
| **Associative** | Grouping of operations doesn't matter |
| **Idempotent**  | Applying same operation twice is safe |

## Performance Optimizations

### 1. Update Batching

Yjs automatically batches rapid edits:

```javascript
// Multiple rapid edits become one update
Y.transact(doc, () => {
  yText.insert(0, "Hello");
  yText.insert(5, " World");
});
```

### 2. State Vector Sync

Instead of sending full state, use state vectors:

```javascript
// Only sync missing updates
const localState = Y.encodeStateVector(doc);
socket.emit("sync-request", { stateVector: localState });

// Server sends only what's missing
socket.on("sync-request", ({ stateVector }) => {
  const update = Y.encodeStateAsUpdate(doc, stateVector);
  socket.emit("y-sync", { update });
});
```

### 3. Garbage Collection

Old deleted content is periodically garbage collected:

```javascript
const doc = new Y.Doc({ gc: true });
```

## Persistence

### Saving Files

When a user saves, the server persists the Yjs document:

```javascript
socket.on("save-file", async ({ filePath }) => {
  const doc = docs.get(filePath);
  const content = doc.getText("monaco").toString();
  await fs.writeFile(filePath, content);
  socket.to(filePath).emit("file-saved", { filePath });
});
```

### Document Recovery

Documents are kept in memory while users are connected. If all users leave:

1. Document is serialized to disk (optional)
2. Or discarded (ephemeral sessions)

## Error Handling

### Disconnection Recovery

```javascript
socket.on("disconnect", () => {
  // Store local state
  const localState = Y.encodeStateAsUpdate(doc);

  // Reconnect with state
  socket.on("connect", () => {
    socket.emit("rejoin-file", { filePath, state: localState });
  });
});
```

### Version Conflicts

If a user's state diverges significantly:

```javascript
socket.on("force-sync", ({ update }) => {
  // Replace local state with server state
  const newDoc = new Y.Doc();
  Y.applyUpdate(newDoc, update);
  // Rebind to editor
});
```

## Security Considerations

### Access Control

- Only users with the session URL can collaborate
- Session IDs are cryptographically random
- Consider adding user authentication for sensitive content

### Rate Limiting

Prevent abuse by limiting:

- Updates per second per client
- Maximum document size
- Maximum collaborators per session

```javascript
const rateLimiter = new RateLimiter({
  windowMs: 1000,
  max: 100, // 100 updates per second
});

socket.on("y-update", (data) => {
  if (rateLimiter.tryAcquire(socket.id)) {
    // Process update
  }
});
```

## Debugging

### Enable Yjs Logging

```javascript
import * as Y from "yjs";

Y.logType = "all"; // Log all operations
```

### Inspect Document State

```javascript
// Dump document content
console.log(doc.getText("monaco").toString());

// Dump internal state
console.log(doc.toJSON());
```

### Socket.IO Debug

```bash
# Enable Socket.IO debugging
DEBUG=socket.io* node server.js
```

## Related Documentation

- [Architecture](./architecture.md) — System overview
- [Execution Engine](./execution-engine.md) — Terminal and file operations
- [Deployment](./deployment.md) — Production setup
