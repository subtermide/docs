# Architecture

This document describes the system architecture of SubTerm, including service breakdown, data flow, and design decisions.

## System Overview

SubTerm follows a microservices architecture where each component has a single responsibility. The system is designed for horizontal scalability, security through isolation, and real-time collaboration.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Client                                       │
│                           (React SPA on :5173)                               │
└──────────────────────────────────────────────────────────────────────────────┘
                    │                                    │
                    │ POST /api/container                │ /workspace/<id>/...
                    ▼                                    ▼
┌─────────────────────────────────┐    ┌─────────────────────────────────────────┐
│           Gateway               │    │               Router                    │
│           (:4000)               │    │              (:5500)                    │
│                                 │    │                                         │
│  • Session management           │    │  • HTTP/WebSocket proxy                 │
│  • Container orchestration      │    │  • Session-to-container routing         │
│  • Resource allocation          │    │  • Activity tracking                    │
└─────────────────────────────────┘    └─────────────────────────────────────────┘
           │                                          │
           │                                          │
           ▼                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  Redis                                          │
│                                 (:6379)                                         │
│                                                                                 │
│   session:<id> → { containerName, lastActive }     container_count → N         │
└─────────────────────────────────────────────────────────────────────────────────┘
           │
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Docker Engine                                      │
│                                                                                 │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                          │
│   │ sess_abc123 │   │ sess_def456 │   │ sess_ghi789 │   ...                    │
│   │  (Server)   │   │  (Server)   │   │  (Server)   │                          │
│   │   :3000     │   │   :3000     │   │   :3000     │                          │
│   └─────────────┘   └─────────────┘   └─────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Service Components

### Client

The browser-based frontend application built with React and Vite.

| Component        | Technology     | Purpose                            |
| ---------------- | -------------- | ---------------------------------- |
| Code Editor      | Monaco Editor  | VS Code-like editing experience    |
| Terminal         | xterm.js       | Terminal emulation in the browser  |
| Collaboration    | Yjs + y-monaco | Real-time document synchronization |
| State Management | Zustand        | Lightweight reactive state         |
| UI Framework     | Material-UI    | Consistent design system           |
| Auth             | Supabase       | User authentication                |

**Key Files:**

- `client/src/pages/WebIDE.jsx` — Main IDE layout and orchestration
- `client/src/components/Terminal.jsx` — Terminal component with PTY connection
- `client/src/hooks/useCollaboration.js` — Yjs collaboration hook
- `client/src/store/useFileStore.js` — File tree state management

### Gateway

The control plane service responsible for container lifecycle management.

**Responsibilities:**

- Create and destroy session containers
- Enforce resource limits (CPU, memory, disk)
- Manage container pool with atomic reservations
- Handle session creation API

**Key Endpoints:**

| Endpoint             | Method | Description                    |
| -------------------- | ------ | ------------------------------ |
| `/api/container`     | POST   | Create a new session container |
| `/api/container/:id` | DELETE | Destroy a session container    |
| `/health`            | GET    | Service health check           |

**Resource Limits (Configurable):**

| Resource             | Default | Environment Variable    |
| -------------------- | ------- | ----------------------- |
| Max Containers       | 10      | `MAX_CONTAINERS`        |
| Memory per Container | 512 MB  | `CONTAINER_MEMORY_MB`   |
| CPU Cores            | 1       | `CONTAINER_CPU_CORES`   |
| Disk Space           | 1 GB    | `CONTAINER_DISK_LIMIT`  |
| Inactivity Timeout   | 10 min  | `INACTIVITY_TIMEOUT_MS` |

### Router

The data plane service that proxies all traffic to session containers.

**Responsibilities:**

- Route HTTP requests to the correct container
- Proxy WebSocket connections (Socket.IO)
- Update session activity timestamps
- Handle load balancing (future)

**Routing Logic:**

```
Request: GET /workspace/abc123/api/fs?path=/
                      ▲
                      │
                Session ID extracted from path
                      │
                      ▼
Redis Lookup: session:abc123 → { containerName: "sess_abc123" }
                      │
                      ▼
Proxy to: http://sess_abc123:3000/api/fs?path=/
```

### Server (Per-Container)

The IDE backend that runs inside each session container.

**Responsibilities:**

- File system operations (CRUD)
- Terminal (PTY) management
- Real-time collaboration document hosting
- File watching and change notifications
- GitHub repository import

**Socket.IO Namespaces:**

| Namespace | Purpose                          |
| --------- | -------------------------------- |
| `/`       | Terminal I/O and file operations |
| `/collab` | Yjs document synchronization     |

**REST Endpoints:**

| Endpoint         | Method | Description                             |
| ---------------- | ------ | --------------------------------------- |
| `/api/fs`        | GET    | List directory contents with git status |
| `/file`          | GET    | Read file content                       |
| `/file`          | POST   | Save file content                       |
| `/api/fs/rename` | POST   | Rename file or directory                |
| `/api/fs/delete` | POST   | Delete file or directory                |
| `/github/import` | POST   | Clone GitHub repository                 |

## Data Flow

### Session Creation Flow

```
┌────────┐     ┌─────────┐     ┌───────┐     ┌────────┐
│ Client │     │ Gateway │     │ Redis │     │ Docker │
└───┬────┘     └────┬────┘     └───┬───┘     └───┬────┘
    │               │              │             │
    │ POST /api/container          │             │
    │──────────────>│              │             │
    │               │              │             │
    │               │ INCR container_count       │
    │               │─────────────>│             │
    │               │              │             │
    │               │ Check < MAX_CONTAINERS     │
    │               │<─────────────│             │
    │               │              │             │
    │               │ Create container           │
    │               │────────────────────────────>
    │               │              │             │
    │               │ SET session:<id>           │
    │               │─────────────>│             │
    │               │              │             │
    │ { sessionId }  │              │             │
    │<──────────────│              │             │
```

### Real-Time Collaboration Flow

```
┌──────────┐     ┌──────────┐     ┌────────┐
│ Client A │     │  Server  │     │Client B│
└────┬─────┘     └────┬─────┘     └───┬────┘
     │                │               │
     │ join-file      │               │
     │───────────────>│               │
     │                │               │
     │ y-sync (state) │               │
     │<───────────────│               │
     │                │               │
     │                │   join-file   │
     │                │<──────────────│
     │                │               │
     │                │ y-sync (state)│
     │                │──────────────>│
     │                │               │
     │ y-update       │               │
     │───────────────>│               │
     │                │               │
     │                │   y-update    │
     │                │──────────────>│
```

## Network Architecture

All services communicate over a Docker bridge network (`subterm-net`), providing DNS-based service discovery.

```
┌─────────────────────────────────────────────────────────────┐
│                     subterm-net (bridge)                    │
│                                                             │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────────┐  │
│   │ gateway │  │ router  │  │  redis  │  │ sess_<id>... │  │
│   │  :4000  │  │  :5500  │  │  :6379  │  │    :3000     │  │
│   └─────────┘  └─────────┘  └─────────┘  └──────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │ Host Network      │
                    │ (exposed ports)   │
                    │ :4000, :5500      │
                    └───────────────────┘
```

## Security Model

### Container Isolation

Each user session runs in a separate Docker container with:

- **Memory limits** — Prevents memory exhaustion attacks
- **CPU limits** — Fair resource sharing
- **PID limits** — Prevents fork bombs (100 processes max)
- **Disk limits** — tmpfs with size restrictions
- **Network isolation** — Containers on private network
- **Auto-removal** — Containers removed on stop

### Request Validation

- Session IDs are cryptographically random (32 bytes hex)
- All paths are validated to prevent directory traversal
- Redis atomic operations prevent race conditions

## Scalability Considerations

### Current Limitations

- Single Gateway instance (can be load-balanced)
- Single Redis instance (can use Redis Cluster)
- Container limit per host (configurable)

### Horizontal Scaling

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │  Host 1  │        │  Host 2  │        │  Host 3  │
   │ Gateway  │        │ Gateway  │        │ Gateway  │
   │ Router   │        │ Router   │        │ Router   │
   │ Containers│       │ Containers│       │ Containers│
   └──────────┘        └──────────┘        └──────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                    ┌────────┴────────┐
                    │  Redis Cluster  │
                    └─────────────────┘
```

## Technology Decisions

| Decision            | Choice    | Rationale                                 |
| ------------------- | --------- | ----------------------------------------- |
| Editor              | Monaco    | Same engine as VS Code, rich API          |
| CRDT Library        | Yjs       | Battle-tested, efficient, Monaco bindings |
| Container Runtime   | Docker    | Industry standard, rich API               |
| Session Store       | Redis     | Fast, atomic operations, pub/sub          |
| Terminal            | node-pty  | Native PTY support, battle-tested         |
| Real-time Transport | Socket.IO | Fallbacks, rooms, namespaces              |

## Related Documentation

- [Deployment Guide](./deployment.md) — How to deploy SubTerm
- [Collaboration](./collaboration.md) — Real-time editing deep dive
- [Execution Engine](./execution-engine.md) — Container and terminal details
