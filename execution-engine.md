# Execution Engine

The execution engine is the core of SubTerm, responsible for container orchestration, terminal execution, and file system operations. This document provides a deep dive into how these systems work.

## Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Execution Engine                                  │
│                                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐  │
│  │   Container     │    │    Terminal     │    │    File System          │  │
│  │ Orchestration   │    │    Execution    │    │    Operations           │  │
│  │                 │    │                 │    │                         │  │
│  │ • Create        │    │ • PTY spawn     │    │ • Read/Write            │  │
│  │ • Destroy       │    │ • I/O streaming │    │ • Watch                 │  │
│  │ • Monitor       │    │ • Resize        │    │ • Git status            │  │
│  │ • Cleanup       │    │ • Shell env     │    │ • GitHub import         │  │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Container Orchestration

### Container Lifecycle

```
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│  Request  │────>│  Reserve  │────>│  Create   │────>│  Running  │
└───────────┘     └───────────┘     └───────────┘     └─────┬─────┘
                                                            │
                                    ┌───────────┐           │
                                    │ Destroyed │<──────────┤
                                    └───────────┘     (timeout/manual)
```

### Session Creation

When a user requests a new session:

```javascript
// 1. Reserve slot atomically
const reserved = await redis.eval(
  `
  local count = redis.call('INCR', 'container_count')
  if count > tonumber(ARGV[1]) then
    redis.call('DECR', 'container_count')
    return 0
  end
  return 1
`,
  0,
  MAX_CONTAINERS,
);

if (!reserved) {
  throw new Error("Container limit reached");
}

// 2. Generate secure session ID
const sessionId = crypto.randomBytes(32).toString("hex");

// 3. Create container
const container = await docker.createContainer({
  Image: "subterm-server",
  name: `sess_${sessionId}`,
  HostConfig: {
    Memory: CONTAINER_MEMORY_MB * 1024 * 1024,
    MemorySwap: CONTAINER_MEMORY_MB * 1024 * 1024, // No swap
    CpuPeriod: 100000,
    CpuQuota: CONTAINER_CPU_CORES * 100000,
    PidsLimit: 100,
    AutoRemove: true,
    NetworkMode: "subterm-net",
  },
  Env: [`SESSION_ID=${sessionId}`],
});

// 4. Start container
await container.start();

// 5. Store session in Redis
await redis.hset(`session:${sessionId}`, {
  containerName: `sess_${sessionId}`,
  lastActive: Date.now(),
});
```

### Resource Limits

Each container runs with strict resource constraints:

| Resource        | Limit        | Purpose                                  |
| --------------- | ------------ | ---------------------------------------- |
| **Memory**      | 512 MB       | Prevent memory exhaustion                |
| **Memory Swap** | 0            | Disable swap for predictable performance |
| **CPU**         | 1 core       | Fair resource sharing                    |
| **PIDs**        | 100          | Prevent fork bombs                       |
| **Disk**        | 1 GB (tmpfs) | Limit workspace size                     |
| **Network**     | Bridge only  | Isolate from host network                |

### Container Configuration

```javascript
const containerConfig = {
  Image: "subterm-server",
  ExposedPorts: { "3000/tcp": {} },
  HostConfig: {
    // Memory limits
    Memory: 512 * 1024 * 1024,
    MemorySwap: 512 * 1024 * 1024,

    // CPU limits
    CpuPeriod: 100000,
    CpuQuota: 100000,

    // Process limits
    PidsLimit: 100,

    // Disk (tmpfs workspace)
    Tmpfs: {
      "/workspace": "size=1G,mode=1777",
    },

    // Auto-cleanup
    AutoRemove: true,

    // Network
    NetworkMode: "subterm-net",
  },
  Env: [`SESSION_ID=${sessionId}`, `WORKSPACE_DIR=/workspace`],
};
```

### Inactivity Cleanup

A background watcher scans for inactive sessions:

```javascript
// timeout.js
const SCAN_INTERVAL = 60000; // 1 minute
const INACTIVITY_TIMEOUT = 600000; // 10 minutes

setInterval(async () => {
  const sessions = await redis.keys("session:*");

  for (const key of sessions) {
    const session = await redis.hgetall(key);
    const idle = Date.now() - parseInt(session.lastActive);

    if (idle > INACTIVITY_TIMEOUT) {
      await destroyContainer(session.containerName);
      await redis.del(key);
      await redis.decr("container_count");
    }
  }
}, SCAN_INTERVAL);
```

### Activity Tracking

Every request updates the session's last activity timestamp:

```javascript
// Router middleware
app.use("/workspace/:sessionId", async (req, res, next) => {
  const { sessionId } = req.params;
  await redis.hset(`session:${sessionId}`, "lastActive", Date.now());
  next();
});
```

## Terminal Execution

### PTY Architecture

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│    Browser       │      │     Server       │      │   Linux PTY      │
│   (xterm.js)     │      │   (node-pty)     │      │    (/bin/bash)   │
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         │  terminal:write         │                         │
         │────────────────────────>│                         │
         │                         │                         │
         │                         │  pty.write(data)        │
         │                         │────────────────────────>│
         │                         │                         │
         │                         │  data (output)          │
         │                         │<────────────────────────│
         │                         │                         │
         │  terminal:data          │                         │
         │<────────────────────────│                         │
```

### PTY Spawning

The server spawns a pseudo-terminal for each session:

```javascript
import * as pty from "node-pty";

const shell = process.env.SHELL || "/bin/bash";

const ptyProcess = pty.spawn(shell, [], {
  name: "xterm-256color",
  cols: 80,
  rows: 24,
  cwd: process.env.WORKSPACE_DIR || "/workspace",
  env: {
    ...process.env,
    TERM: "xterm-256color",
    COLORTERM: "truecolor",
  },
});
```

### Terminal Events

| Event             | Direction       | Description                 |
| ----------------- | --------------- | --------------------------- |
| `terminal:data`   | Server → Client | Output from PTY             |
| `terminal:write`  | Client → Server | Input to PTY                |
| `terminal:resize` | Client → Server | Terminal dimensions changed |

### Event Handlers

```javascript
// Server-side
socket.on("terminal:write", (data) => {
  ptyProcess.write(data);
});

socket.on("terminal:resize", ({ cols, rows }) => {
  ptyProcess.resize(cols, rows);
});

ptyProcess.onData((data) => {
  socket.emit("terminal:data", data);
});

ptyProcess.onExit(({ exitCode }) => {
  socket.emit("terminal:exit", { exitCode });
});
```

### Client-Side Terminal

```javascript
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";

const terminal = new Terminal({
  cursorBlink: true,
  fontSize: 14,
  fontFamily: "Menlo, Monaco, monospace",
  theme: {
    background: "#1e1e1e",
    foreground: "#d4d4d4",
  },
});

const fitAddon = new FitAddon();
terminal.loadAddon(fitAddon);
terminal.open(containerRef.current);
fitAddon.fit();

// Handle input
terminal.onData((data) => {
  socket.emit("terminal:write", data);
});

// Handle output
socket.on("terminal:data", (data) => {
  terminal.write(data);
});

// Handle resize
const resizeObserver = new ResizeObserver(() => {
  fitAddon.fit();
  socket.emit("terminal:resize", {
    cols: terminal.cols,
    rows: terminal.rows,
  });
});
```

## File System Operations

### API Endpoints

| Endpoint         | Method | Description                    |
| ---------------- | ------ | ------------------------------ |
| `/api/fs`        | GET    | List directory with git status |
| `/file`          | GET    | Read file content              |
| `/file`          | POST   | Write file content             |
| `/api/fs/rename` | POST   | Rename file or directory       |
| `/api/fs/delete` | POST   | Delete file or directory       |
| `/github/import` | POST   | Clone GitHub repository        |

### Directory Listing

```javascript
app.get("/api/fs", async (req, res) => {
  const dirPath = req.query.path || "/workspace";
  const entries = await fs.readdir(dirPath, { withFileTypes: true });

  const results = await Promise.all(
    entries.map(async (entry) => {
      const fullPath = path.join(dirPath, entry.name);
      const stats = await fs.stat(fullPath);

      return {
        name: entry.name,
        path: fullPath,
        type: entry.isDirectory() ? "directory" : "file",
        size: stats.size,
        modified: stats.mtime,
        gitStatus: await getGitStatus(fullPath),
      };
    }),
  );

  res.json(results);
});
```

### Git Status Integration

```javascript
async function getGitStatus(filePath) {
  try {
    const { stdout } = await exec(`git status --porcelain "${filePath}"`, {
      cwd: path.dirname(filePath),
    });

    const status = stdout.trim().slice(0, 2);
    return (
      {
        "M ": "modified",
        "A ": "added",
        "D ": "deleted",
        "??": "untracked",
        " M": "modified",
        "": "clean",
      }[status] || "unknown"
    );
  } catch {
    return null; // Not in a git repo
  }
}
```

### File Reading

```javascript
app.get("/file", async (req, res) => {
  const filePath = req.query.path;

  // Security: Validate path is within workspace
  const resolved = path.resolve(filePath);
  if (!resolved.startsWith("/workspace")) {
    return res.status(403).json({ error: "Access denied" });
  }

  const content = await fs.readFile(filePath, "utf-8");
  res.json({ content });
});
```

### File Writing

```javascript
app.post("/file", async (req, res) => {
  const { path: filePath, content } = req.body;

  // Check if file is being collaboratively edited
  const doc = collabDocs.get(filePath);
  if (doc) {
    // Update Yjs document
    const yText = doc.getText("monaco");
    doc.transact(() => {
      yText.delete(0, yText.length);
      yText.insert(0, content);
    });
  }

  await fs.writeFile(filePath, content, "utf-8");
  res.json({ success: true });
});
```

### File Watching

Real-time file system watching with chokidar:

```javascript
import chokidar from "chokidar";

const watcher = chokidar.watch("/workspace", {
  ignored: ["**/node_modules/**", "**/.git/**", "**/*.swp", "**/*.tmp"],
  persistent: true,
  ignoreInitial: true,
  awaitWriteFinish: {
    stabilityThreshold: 100,
    pollInterval: 50,
  },
});

watcher.on("all", (event, filePath) => {
  io.emit("fs-event", {
    type: event, // add, change, unlink, addDir, unlinkDir
    path: filePath,
  });
});
```

### GitHub Import

```javascript
app.post("/github/import", async (req, res) => {
  const { repoUrl } = req.body;

  // Validate URL
  const match = repoUrl.match(/github\.com\/([^\/]+)\/([^\/]+)/);
  if (!match) {
    return res.status(400).json({ error: "Invalid GitHub URL" });
  }

  const [, owner, repo] = match;
  const cloneUrl = `https://github.com/${owner}/${repo}.git`;
  const targetDir = `/workspace/${repo}`;

  try {
    await exec(`git clone --depth 1 ${cloneUrl} ${targetDir}`);
    res.json({ success: true, path: targetDir });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## Docker API Integration

### Dockerode Client

```javascript
import Docker from "dockerode";

const docker = new Docker({
  socketPath: "/var/run/docker.sock",
});
```

### Container Operations

```javascript
// List running session containers
async function listSessions() {
  const containers = await docker.listContainers({
    filters: { name: ["sess_"] },
  });
  return containers;
}

// Get container stats
async function getContainerStats(containerId) {
  const container = docker.getContainer(containerId);
  const stats = await container.stats({ stream: false });
  return {
    cpu: calculateCpuPercent(stats),
    memory: stats.memory_stats.usage,
    network: stats.networks,
  };
}

// Stop container
async function destroyContainer(containerName) {
  const container = docker.getContainer(containerName);
  await container.stop();
  // AutoRemove handles cleanup
}
```

### Container Events

Monitor container events for debugging:

```javascript
docker.getEvents({}, (err, stream) => {
  stream.on("data", (chunk) => {
    const event = JSON.parse(chunk.toString());
    if (event.Actor?.Attributes?.name?.startsWith("sess_")) {
      console.log(`Container ${event.Action}: ${event.Actor.Attributes.name}`);
    }
  });
});
```

## Error Handling

### Container Failures

```javascript
container.on("die", async (data) => {
  const exitCode = data.exitCode;
  console.error(`Container died with code ${exitCode}`);

  // Cleanup Redis
  await redis.del(`session:${sessionId}`);
  await redis.decr("container_count");

  // Notify clients
  io.to(sessionId).emit("session:terminated", {
    reason: "container_crashed",
    exitCode,
  });
});
```

### PTY Crashes

```javascript
ptyProcess.onExit(({ exitCode, signal }) => {
  if (exitCode !== 0) {
    console.error(`PTY exited with code ${exitCode}, signal ${signal}`);

    // Respawn after delay
    setTimeout(() => {
      spawnPty();
    }, 1000);
  }
});
```

### File System Errors

```javascript
app.use((err, req, res, next) => {
  if (err.code === "ENOENT") {
    return res.status(404).json({ error: "File not found" });
  }
  if (err.code === "EACCES") {
    return res.status(403).json({ error: "Permission denied" });
  }
  if (err.code === "ENOSPC") {
    return res.status(507).json({ error: "Disk quota exceeded" });
  }
  next(err);
});
```

## Performance Optimization

### Connection Pooling

```javascript
// Redis connection pool
const redis = new Redis({
  host: "redis",
  port: 6379,
  maxRetriesPerRequest: 3,
  retryDelayOnFailover: 100,
});
```

### Buffered Terminal Output

```javascript
// Batch terminal output
let buffer = "";
let flushTimeout = null;

ptyProcess.onData((data) => {
  buffer += data;

  if (!flushTimeout) {
    flushTimeout = setTimeout(() => {
      socket.emit("terminal:data", buffer);
      buffer = "";
      flushTimeout = null;
    }, 10); // 10ms batching
  }
});
```

### File Caching

```javascript
// LRU cache for recently accessed files
import LRU from "lru-cache";

const fileCache = new LRU({
  max: 100,
  maxAge: 60000, // 1 minute
});

app.get("/file", async (req, res) => {
  const cached = fileCache.get(req.query.path);
  if (cached) {
    return res.json({ content: cached });
  }

  const content = await fs.readFile(req.query.path, "utf-8");
  fileCache.set(req.query.path, content);
  res.json({ content });
});
```

## Security

### Path Traversal Prevention

```javascript
function sanitizePath(userPath) {
  const resolved = path.resolve("/workspace", userPath);
  if (!resolved.startsWith("/workspace")) {
    throw new Error("Path traversal detected");
  }
  return resolved;
}
```

### Command Injection Prevention

```javascript
// Never interpolate user input into shell commands
// BAD
exec(`git clone ${userUrl}`);

// GOOD
execFile("git", ["clone", userUrl]);
```

### Resource Exhaustion Prevention

```javascript
// Limit concurrent file operations
const semaphore = new Semaphore(10);

app.get("/file", async (req, res) => {
  await semaphore.acquire();
  try {
    const content = await fs.readFile(req.query.path);
    res.json({ content });
  } finally {
    semaphore.release();
  }
});
```

## Related Documentation

- [Architecture](./architecture.md) — System design overview
- [Collaboration](./collaboration.md) — Real-time editing
- [Deployment](./deployment.md) — Production setup
