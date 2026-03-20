# Deployment

This guide covers how to deploy SubTerm in development and production environments.

## Prerequisites

Before deploying SubTerm, ensure you have the following installed:

| Requirement    | Minimum Version | Purpose               |
| -------------- | --------------- | --------------------- |
| Node.js        | 20.x            | Server runtime        |
| pnpm           | 8.x             | Package manager       |
| Docker         | 24.x            | Container runtime     |
| Docker Compose | 2.x             | Service orchestration |
| Redis          | 7.x             | Session store         |

## Quick Start (Development)

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/subterm.git
cd subterm
```

### 2. Start Infrastructure Services

```bash
cd gateway
docker compose up -d
```

This starts Redis and creates the Docker network.

### 3. Build the Server Image

```bash
cd ../server
docker build -t subterm-server .
```

### 4. Start the Gateway

```bash
cd ../gateway
pnpm install
pnpm dev
```

### 5. Start the Router

```bash
cd ../router
pnpm install
pnpm dev
```

### 6. Start the Client

```bash
cd ../client
pnpm install
pnpm dev
```

The application is now available at `http://localhost:5173`.

## Production Deployment

### Docker Images

Build all required Docker images:

```bash
# Server (runs inside each session container)
docker build -t subterm-server ./server

# Gateway (container orchestration)
docker build -t subterm-gateway ./gateway

# Router (traffic proxy)
docker build -t subterm-router ./router
```

### Docker Compose (Full Stack)

Create a production `docker-compose.yml`:

```yaml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    networks:
      - subterm-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  gateway:
    image: subterm-gateway
    restart: unless-stopped
    ports:
      - "4000:4000"
    environment:
      - REDIS_URL=redis://redis:6379
      - MAX_CONTAINERS=50
      - CONTAINER_MEMORY_MB=512
      - CONTAINER_CPU_CORES=1
      - CONTAINER_DISK_LIMIT=1G
      - INACTIVITY_TIMEOUT_MS=600000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - subterm-net
    depends_on:
      redis:
        condition: service_healthy

  router:
    image: subterm-router
    restart: unless-stopped
    ports:
      - "5500:5500"
    environment:
      - REDIS_URL=redis://redis:6379
    networks:
      - subterm-net
    depends_on:
      redis:
        condition: service_healthy

networks:
  subterm-net:
    name: subterm-net
    driver: bridge

volumes:
  redis-data:
```

Deploy with:

```bash
docker compose -f docker-compose.prod.yml up -d
```

### Client Build

Build the static frontend assets:

```bash
cd client
pnpm install
pnpm build
```

The built files are in `client/dist/`. Serve them with Nginx, Caddy, or any static file server.

### Nginx Configuration

```nginx
upstream gateway {
    server localhost:4000;
}

upstream router {
    server localhost:5500;
}

server {
    listen 80;
    server_name subterm.example.com;

    # Static frontend
    location / {
        root /var/www/subterm/dist;
        try_files $uri $uri/ /index.html;
    }

    # Gateway API
    location /api/container {
        proxy_pass http://gateway;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Workspace API (Router)
    location /workspace/ {
        proxy_pass http://router;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 86400;
    }
}
```

## Environment Variables

### Gateway

| Variable                | Default                  | Description                           |
| ----------------------- | ------------------------ | ------------------------------------- |
| `PORT`                  | `4000`                   | Gateway HTTP port                     |
| `REDIS_URL`             | `redis://localhost:6379` | Redis connection string               |
| `MAX_CONTAINERS`        | `10`                     | Maximum concurrent session containers |
| `CONTAINER_MEMORY_MB`   | `512`                    | Memory limit per container (MB)       |
| `CONTAINER_CPU_CORES`   | `1`                      | CPU cores per container               |
| `CONTAINER_DISK_LIMIT`  | `1G`                     | Disk space per container              |
| `INACTIVITY_TIMEOUT_MS` | `600000`                 | Session timeout (10 min default)      |
| `DOCKER_NETWORK`        | `subterm-net`            | Docker network name                   |
| `SERVER_IMAGE`          | `subterm-server`         | Server Docker image name              |

### Router

| Variable    | Default                  | Description             |
| ----------- | ------------------------ | ----------------------- |
| `PORT`      | `5500`                   | Router HTTP port        |
| `REDIS_URL` | `redis://localhost:6379` | Redis connection string |

### Client

| Variable                 | Default                 | Description            |
| ------------------------ | ----------------------- | ---------------------- |
| `VITE_API`               | `http://localhost:5500` | Router URL             |
| `VITE_GATEWAY_URL`       | `http://localhost:4000` | Gateway URL            |
| `VITE_SUPABASE_URL`      | —                       | Supabase project URL   |
| `VITE_SUPABASE_ANON_KEY` | —                       | Supabase anonymous key |

Create a `.env` file for the client:

```bash
# client/.env.production
VITE_API=https://api.subterm.example.com
VITE_GATEWAY_URL=https://gateway.subterm.example.com
VITE_SUPABASE_URL=https://xxx.supabase.co
VITE_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIs...
```

## Scaling

### Vertical Scaling

Increase resources per container:

```bash
MAX_CONTAINERS=100
CONTAINER_MEMORY_MB=1024
CONTAINER_CPU_CORES=2
```

### Horizontal Scaling

Deploy multiple instances behind a load balancer:

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │   (HAProxy)     │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────┴────┐         ┌────┴────┐         ┌────┴────┐
    │ Host 1  │         │ Host 2  │         │ Host 3  │
    │ Gateway │         │ Gateway │         │ Gateway │
    │ Router  │         │ Router  │         │ Router  │
    └─────────┘         └─────────┘         └─────────┘
                             │
                    ┌────────┴────────┐
                    │  Redis Cluster  │
                    └─────────────────┘
```

**HAProxy Configuration:**

```haproxy
frontend http
    bind *:80
    default_backend routers

backend routers
    balance roundrobin
    option httpchk GET /health
    server router1 host1:5500 check
    server router2 host2:5500 check
    server router3 host3:5500 check
```

## Monitoring

### Health Checks

All services expose health endpoints:

| Service | Endpoint      | Expected Response |
| ------- | ------------- | ----------------- |
| Gateway | `GET /health` | `200 OK`          |
| Router  | `GET /health` | `200 OK`          |
| Server  | `GET /health` | `200 OK`          |

### Metrics

Monitor these key metrics:

| Metric                | Source                  | Alert Threshold |
| --------------------- | ----------------------- | --------------- |
| Active Containers     | Redis `container_count` | > 80% of MAX    |
| Memory Usage          | Docker stats            | > 90%           |
| Response Time         | Load balancer           | p99 > 500ms     |
| WebSocket Connections | Router                  | > 10k           |

### Logging

All services output JSON logs to stdout. Use a log aggregator like Loki or ELK:

```bash
# View gateway logs
docker logs -f subterm-gateway

# View all container logs
docker compose logs -f
```

## Security Hardening

### Docker Socket Security

The gateway requires access to the Docker socket. In production:

1. Use Docker socket proxy (e.g., Tecnativa/docker-socket-proxy)
2. Limit API access to required endpoints only

```yaml
docker-socket-proxy:
  image: tecnativa/docker-socket-proxy
  environment:
    - CONTAINERS=1
    - NETWORKS=1
    - POST=1
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
```

### Network Security

- Use TLS for all external connections
- Keep Redis on internal network only
- Use firewall rules to restrict access

### Container Security

Containers run with restricted capabilities:

- No privileged mode
- Limited syscalls via seccomp
- Read-only root filesystem (optional)
- Non-root user inside container

## Backup & Recovery

### Redis Persistence

Enable Redis persistence for session recovery:

```conf
# redis.conf
appendonly yes
appendfsync everysec
```

### Disaster Recovery

1. **Session loss** — Users can create new sessions; code is ephemeral
2. **Redis failure** — Sessions are lost; restart services
3. **Docker failure** — Orphan containers auto-cleaned on restart

## Troubleshooting

### Container Not Starting

```bash
# Check gateway logs
docker logs subterm-gateway

# Check Docker events
docker events --filter 'type=container'

# Verify network exists
docker network ls | grep subterm-net
```

### WebSocket Connection Failed

```bash
# Test router connectivity
curl -i http://localhost:5500/health

# Check for firewall issues
sudo iptables -L -n | grep 5500
```

### Session Not Found

```bash
# Check Redis for session
redis-cli KEYS "session:*"

# Verify container is running
docker ps | grep sess_
```

## Related Documentation

- [Architecture](./architecture.md) — System design overview
- [Collaboration](./collaboration.md) — Real-time editing setup
- [Execution Engine](./execution-engine.md) — Container details
