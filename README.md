# SubTerm Documentation

Welcome to the official documentation for **SubTerm** — a cloud-based integrated development environment with real-time collaboration capabilities.

## What is SubTerm?

SubTerm is a browser-based IDE that provides developers with a fully-featured coding environment accessible from anywhere. Each user session runs in an isolated Docker container, ensuring security and resource separation while enabling powerful features like terminal access, file management, and collaborative editing.

## Key Features

| Feature                     | Description                                                       |
| --------------------------- | ----------------------------------------------------------------- |
| **Monaco Editor**           | VS Code-powered editor with syntax highlighting for 20+ languages |
| **Integrated Terminal**     | Full bash shell access with PTY support                           |
| **Real-time Collaboration** | Edit code simultaneously with teammates using CRDT-based sync     |
| **Sandboxed Containers**    | Each session runs in an isolated Docker container                 |
| **GitHub Import**           | Clone public repositories directly into your workspace            |
| **File Management**         | Create, rename, delete files with Git status integration          |
| **Session Sharing**         | Invite collaborators to join your workspace instantly             |

## Documentation Index

| Document                                  | Description                                                      |
| ----------------------------------------- | ---------------------------------------------------------------- |
| [Architecture](./architecture.md)         | System design, service breakdown, and data flow                  |
| [Deployment](./deployment.md)             | Installation, configuration, and production setup                |
| [Collaboration](./collaboration.md)       | Real-time editing, Yjs integration, and session sharing          |
| [Execution Engine](./execution-engine.md) | Container orchestration, terminal execution, and file operations |

## Quick Start

```bash
# Clone the repository
git clone https://github.com/your-org/subterm.git
cd subterm

# Start infrastructure services
cd gateway && docker compose up -d

# Build and run the client
cd ../client && pnpm install && pnpm dev
```

For detailed setup instructions, see the [Deployment Guide](./deployment.md).

## Tech Stack

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend                            │
│  React 18 • Vite • Monaco Editor • xterm.js • Yjs • MUI    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Infrastructure                         │
│         Gateway • Router • Redis • Docker Engine            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Session Containers                       │
│     Node.js Server • Socket.IO • node-pty • chokidar       │
└─────────────────────────────────────────────────────────────┘
```

## Architecture Overview

SubTerm uses a microservices architecture with three core services:

- **Gateway** — Handles session creation, container lifecycle, and authentication
- **Router** — Proxies HTTP/WebSocket traffic to the appropriate container
- **Server** — Runs inside each container, providing IDE functionality

See [Architecture](./architecture.md) for detailed diagrams and explanations.

## Contributing

We welcome contributions! Please read our contributing guidelines before submitting pull requests.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License — see the LICENSE file for details.

---

<p align="center">
  Built with ❤️ by the SubTerm Team
</p>
