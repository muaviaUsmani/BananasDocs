---
sidebar_position: 1
---

# Installation

This guide covers multiple ways to install and run Bananas, from quick Docker setup to local development.

## Prerequisites

### Required
- **Docker** 20.10+ and **Docker Compose** 2.0+
  - [Install Docker Desktop](https://docs.docker.com/get-docker/) (includes Compose)
  - Verify: `docker --version && docker compose version`

### Optional
- **Go** 1.21+ for local development without Docker
  - [Download Go](https://golang.org/dl/)
  - Verify: `go version`

- **Redis** 6.2+ if running services locally
  - [Install Redis](https://redis.io/download)
  - Or use Docker: `docker run -d -p 6379:6379 redis:7-alpine`

## Quick Install (Docker)

The fastest way to get Bananas running:

### 1. Clone the Repository

```bash
git clone https://github.com/muaviaUsmani/Bananas.git
cd Bananas
```

### 2. Start Development Environment

```bash
make dev
```

This command:
- Pulls necessary Docker images
- Starts Redis, API server, workers, and scheduler
- Enables hot reload for code changes
- Exposes API on `localhost:8080`

### 3. Verify Installation

```bash
# Check all services are running
docker compose -f docker-compose.dev.yml ps

# Test the API
curl http://localhost:8080/health
```

Expected output: `{"status": "healthy"}`

## Production Install (Docker)

For staging or production environments:

### 1. Build Production Images

```bash
make prod-build
```

This creates optimized binaries with:
- Multi-stage Docker builds
- Minimal alpine-based images
- No development dependencies

### 2. Start Production Services

```bash
docker compose up -d
```

Services started:
- **Redis** (port 6379) - With persistence enabled
- **API Server** (port 8080) - RESTful job submission
- **3x Workers** - Process jobs from queues
- **Scheduler** - Handles scheduled and retry jobs

### 3. Scale Workers

```bash
make scale-workers N=10
```

This scales the worker service to 10 instances for higher throughput.

## Local Development Install

For Go developers who want to run services natively:

### 1. Install Dependencies

```bash
# Install Go dependencies
go mod download

# Start Redis (required)
docker run -d -p 6379:6379 redis:7-alpine
# Or use local Redis: redis-server
```

### 2. Build Binaries

```bash
# Build all services
make build

# Binaries created in ./bin/
# - api
# - worker
# - scheduler
```

### 3. Run Services

```bash
# Terminal 1: API Server
./bin/api

# Terminal 2: Worker
./bin/worker

# Terminal 3: Scheduler
./bin/scheduler
```

## Environment Configuration

### Development (.env.dev)

```bash
REDIS_URL=redis://localhost:6379
API_PORT=8080
WORKER_CONCURRENCY=5
LOG_LEVEL=debug
```

### Production (.env.prod)

```bash
REDIS_URL=redis://:password@redis-prod:6379
API_PORT=8080
WORKER_CONCURRENCY=20
LOG_LEVEL=info
RESULT_TTL_SUCCESS=3600
RESULT_TTL_FAILURE=86400
```

Load environment files:

```bash
# Development
docker compose --env-file .env.dev up

# Production
docker compose --env-file .env.prod up
```

## Verify Installation

### Submit a Test Job

```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test_job",
    "payload": {"message": "Hello Bananas!"},
    "priority": "normal"
  }'
```

### Check Job Status

```bash
# Save the job ID from previous response
JOB_ID="<returned-job-id>"

curl http://localhost:8080/jobs/$JOB_ID
```

### View Logs

```bash
# All services
make logs

# Specific service
docker compose logs -f worker

# Tail last 100 lines
docker compose logs --tail=100 api
```

## Common Installation Issues

### Port Already in Use

**Error**: `Bind for 0.0.0.0:8080 failed: port is already allocated`

**Solution**:
```bash
# Find process using port 8080
lsof -i :8080

# Kill the process or change the port
API_PORT=8081 docker compose up
```

### Redis Connection Failed

**Error**: `dial tcp 127.0.0.1:6379: connect: connection refused`

**Solution**:
```bash
# Check Redis is running
docker ps | grep redis

# Restart Redis
docker compose restart redis

# Check Redis logs
docker compose logs redis
```

### Permission Denied

**Error**: `permission denied while trying to connect to Docker daemon`

**Solution**:
```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Log out and back in, or run:
newgrp docker
```

## Next Steps

Now that Bananas is installed:

- **[Quick Start Guide](./quickstart)** - Submit your first real job
- **[Testing](./testing)** - Run the test suite
- **[Core Concepts](../category/core-concepts)** - Understanding how it works
