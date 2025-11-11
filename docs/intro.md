---
sidebar_position: 1
---

# Welcome to Bananas

Welcome to **Bananas** - a lightweight, scalable distributed task queue built with Go, Redis, and Docker for handling asynchronous job processing across multiple workers.

## What is Bananas?

Bananas is a production-ready distributed task queue system that enables you to:

- **Process jobs asynchronously** across multiple worker instances
- **Handle priority-based queuing** with High, Normal, and Low priorities
- **Scale horizontally** by adding more workers as your load increases
- **Route tasks intelligently** to specialized worker pools
- **Retry failed jobs** automatically with exponential backoff
- **Schedule jobs** for future execution
- **Monitor system health** with built-in metrics

## Why Choose Bananas?

### High Performance
- **1,650+ jobs/second** with 20 workers
- **Sub-3ms p99 latency** for job submission
- Lightweight goroutine-based concurrency
- Efficient Redis-backed queue operations

### Production Ready
- **93.3% test coverage** across all components
- Automatic retry with exponential backoff
- Dead letter queue for failed jobs
- Graceful shutdown handling
- Built-in result backend with TTL

### Easy to Deploy
- Docker and Docker Compose support
- Kubernetes-ready with Helm charts
- Systemd service templates
- Hot reload development mode

## Quick Start

Get Bananas running in under 2 minutes:

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- (Optional) [Go 1.21+](https://golang.org/dl/) for local development

### Start Development Environment

```bash
# Clone the repository
git clone https://github.com/muaviaUsmani/Bananas.git
cd Bananas

# Start all services with hot reload
make dev
```

This starts:
- Redis (port 6379)
- API Server (port 8080)
- 3 Worker instances
- Scheduler service

### Submit Your First Job

```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "send_email",
    "payload": {"to": "user@example.com", "subject": "Hello!"},
    "priority": "high"
  }'
```

### Check Job Status

```bash
curl http://localhost:8080/jobs/{job-id}
```

## Core Features

### Priority Queuing
Process high-priority jobs first while ensuring lower-priority jobs aren't starved.

### Task Routing
Direct jobs to specialized worker pools (GPU workers, email workers, etc.) for resource isolation.

### Automatic Retries
Failed jobs retry automatically with exponential backoff (2s, 4s, 8s...) to give failing dependencies time to recover.

### Scheduled Jobs
Submit jobs for future execution or implement periodic tasks with cron expressions.

### Result Backend
Store and retrieve job results with configurable TTL for successful and failed jobs.

### Graceful Shutdown
Workers complete in-flight jobs before shutting down, preventing job loss during deployments.

## Architecture Highlights

Bananas uses a simple but powerful architecture:

- **Redis** as the message broker and data store
- **Stateless workers** enabling easy horizontal scaling
- **Atomic operations** ensuring no job loss
- **Goroutine-based concurrency** for lightweight parallelism

## Next Steps

Explore these resources to get the most out of Bananas:

- **[Getting Started](category/getting-started)** - Detailed installation and setup guide
- **[Core Concepts](category/core-concepts)** - Understand how Bananas works
- **[API Reference](category/api-reference)** - Complete API documentation
- **[Deployment Guide](guides/deployment)** - Production deployment patterns
- **[Troubleshooting](guides/troubleshooting)** - Common issues and solutions

## Community & Support

Need help or want to contribute?

- **GitHub**: [muaviaUsmani/Bananas](https://github.com/muaviaUsmani/Bananas)
- **Issues**: Report bugs or request features
- **Discussions**: Ask questions and share ideas
- **Contributing**: See our [Contributing Guide](https://github.com/muaviaUsmani/Bananas/blob/main/CONTRIBUTING.md)

## License

Bananas is open source software licensed under the [MIT License](https://github.com/muaviaUsmani/Bananas/blob/main/LICENSE).

---

Ready to build scalable, reliable job processing systems? Let's dive in! 🍌
