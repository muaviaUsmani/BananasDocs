---
sidebar_position: 2
---

# Quick Start

Get hands-on with Bananas in 5 minutes. This guide walks you through submitting jobs, checking status, and seeing results.

## Prerequisites

Make sure Bananas is [installed and running](./installation):

```bash
# Quick check
curl http://localhost:8080/health
# Should return: {"status":"healthy"}
```

## Your First Job

### 1. Submit a Simple Job

```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "send_email",
    "payload": {
      "to": "user@example.com",
      "subject": "Welcome!",
      "body": "Thanks for trying Bananas!"
    },
    "priority": "normal"
  }'
```

**Response**:
```json
{
  "id": "job_abc123def456",
  "status": "pending",
  "created_at": "2024-01-15T10:30:00Z"
}
```

Save the `id` - you'll need it to check status!

### 2. Check Job Status

```bash
curl http://localhost:8080/jobs/job_abc123def456
```

**Response**:
```json
{
  "id": "job_abc123def456",
  "name": "send_email",
  "status": "completed",
  "priority": "normal",
  "payload": {...},
  "result": {
    "success": true,
    "data": "Email sent successfully"
  },
  "attempts": 1,
  "created_at": "2024-01-15T10:30:00Z",
  "completed_at": "2024-01-15T10:30:02Z"
}
```

## Priority Jobs

### High Priority Job

High priority jobs jump to the front of the queue:

```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "urgent_notification",
    "payload": {"user_id": 12345, "message": "Critical alert!"},
    "priority": "high",
    "description": "Send critical system alert"
  }'
```

Priority order: **High** → **Normal** → **Low**

### Low Priority Job

Perfect for background tasks that aren't time-sensitive:

```bash
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "generate_report",
    "payload": {"report_type": "monthly", "month": "December"},
    "priority": "low"
  }'
```

## Scheduled Jobs

### Schedule for Future Execution

```bash
# Schedule job for 1 hour from now
SCHEDULED_TIME=$(date -u -d "+1 hour" +"%Y-%m-%dT%H:%M:%SZ")

curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"send_reminder\",
    \"payload\": {\"user_id\": 789, \"message\": \"Don't forget!\"},
    \"priority\": \"normal\",
    \"scheduled_for\": \"$SCHEDULED_TIME\"
  }"
```

The job won't execute until the scheduled time arrives.

## Task Routing

### Route to Specialized Workers

Direct jobs to specific worker pools:

```bash
# GPU-intensive job
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "train_model",
    "payload": {"model": "resnet50", "dataset": "imagenet"},
    "priority": "high",
    "routing_key": "gpu"
  }'

# Email-specific job
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "send_bulk_email",
    "payload": {"campaign_id": 456},
    "priority": "normal",
    "routing_key": "email"
  }'
```

Workers must be configured with matching routing keys to process these jobs.

## Using the Go Client SDK

### Install the Client

```bash
go get github.com/muaviaUsmani/bananas/pkg/client
```

### Submit Jobs from Go

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/muaviaUsmani/bananas/pkg/client"
)

func main() {
    // Create client
    c := client.NewClient("redis://localhost:6379")
    defer c.Close()

    // Submit a job
    jobID, err := c.SubmitJob(
        "process_order",
        map[string]interface{}{
            "order_id": 12345,
            "items": []string{"item1", "item2"},
        },
        client.PriorityNormal,
        "Process customer order",
    )
    if err != nil {
        panic(err)
    }

    fmt.Printf("Job submitted: %s\n", jobID)

    // Wait for result (blocks until complete or timeout)
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    result, err := c.GetResult(ctx, jobID)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Job completed: %+v\n", result)
}
```

### Submit and Wait (RPC-Style)

```go
// Submit job and block until result is ready
ctx := context.WithTimeout(context.Background(), 1*time.Minute)

result, err := c.SubmitAndWait(
    ctx,
    "calculate_total",
    map[string]interface{}{"items": [1, 2, 3, 4, 5]},
    client.PriorityHigh,
    30*time.Second,
)

if err != nil {
    log.Fatal(err)
}

fmt.Printf("Result: %v\n", result.Data)
```

## Monitoring Jobs

### List All Jobs

```bash
# Get recent jobs (last 100)
curl http://localhost:8080/jobs?limit=100
```

### Filter by Status

```bash
# Pending jobs
curl http://localhost:8080/jobs?status=pending

# Failed jobs
curl http://localhost:8080/jobs?status=failed

# Processing jobs
curl http://localhost:8080/jobs?status=processing
```

### Queue Metrics

```bash
curl http://localhost:8080/metrics
```

**Response**:
```json
{
  "queues": {
    "high": 5,
    "normal": 142,
    "low": 23
  },
  "processing": 12,
  "scheduled": 8,
  "dead_letter": 2,
  "workers": {
    "active": 10,
    "utilization": 0.75
  }
}
```

## Common Patterns

### Batch Job Submission

```bash
# Submit multiple jobs at once
for i in {1..10}; do
  curl -X POST http://localhost:8080/jobs \
    -H "Content-Type: application/json" \
    -d "{
      \"name\": \"process_item\",
      \"payload\": {\"item_id\": $i},
      \"priority\": \"normal\"
    }"
done
```

### Job Chaining (Manual)

```go
// Submit first job
job1ID, _ := c.SubmitJob("step1", payload1, client.PriorityNormal, "First step")

// Wait for completion
result1, _ := c.GetResult(ctx, job1ID)

// Submit dependent job
job2ID, _ := c.SubmitJob("step2", result1.Data, client.PriorityNormal, "Second step")
```

### Retry Failed Jobs

```bash
# Get job details
JOB_ID="job_failed123"
curl http://localhost:8080/jobs/$JOB_ID

# Resubmit with same payload
curl -X POST http://localhost:8080/jobs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "send_email",
    "payload": {...same payload...},
    "priority": "high"
  }'
```

## What's Next?

Now that you've submitted jobs, dive deeper:

- **[Core Concepts](../category/core-concepts)** - Understand job lifecycle, queues, and workers
- **[API Reference](../category/api-reference)** - Complete API documentation
- **[Deployment](../guides/deployment)** - Deploy to production
- **[Troubleshooting](../guides/troubleshooting)** - Common issues and solutions

## Quick Reference

### Job Priorities
- `high` - Processed first
- `normal` - Default priority
- `low` - Background tasks

### Job Statuses
- `pending` - Waiting in queue
- `processing` - Currently executing
- `completed` - Successfully finished
- `failed` - Execution failed
- `scheduled` - Waiting for scheduled time

### API Endpoints
- `POST /jobs` - Submit job
- `GET /jobs/:id` - Get job status
- `GET /jobs` - List jobs
- `GET /metrics` - Queue metrics
- `GET /health` - Health check
