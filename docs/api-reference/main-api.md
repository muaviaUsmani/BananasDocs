---
sidebar_position: 1
---

# Client SDK API

Complete reference for the Bananas Go client library.

## Installation

```bash
go get github.com/muaviaUsmani/bananas/pkg/client
```

## Quick Example

```go
package main

import (
    "context"
    "log"
    "time"

    "github.com/muaviaUsmani/bananas/pkg/client"
)

func main() {
    // Create client
    c := client.NewClient("redis://localhost:6379")
    defer c.Close()

    // Submit job
    jobID, err := c.SubmitJob(
        "send_email",
        map[string]interface{}{
            "to": "user@example.com",
            "subject": "Hello!",
        },
        client.PriorityHigh,
        "Send welcome email",
    )
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Job submitted: %s", jobID)
}
```

## Client Creation

### NewClient

Creates a new client with default configuration.

```go
func NewClient(redisURL string) *Client
```

**Parameters:**
- `redisURL` (string) - Redis connection URL

**Example:**
```go
client := client.NewClient("redis://localhost:6379")
client := client.NewClient("redis://:password@prod-redis:6379")
```

### NewClientWithConfig

Creates a client with custom result TTL configuration.

```go
func NewClientWithConfig(redisURL string, successTTL, failureTTL time.Duration) *Client
```

**Parameters:**
- `redisURL` - Redis connection URL
- `successTTL` - TTL for successful job results
- `failureTTL` - TTL for failed job results

**Example:**
```go
client := client.NewClientWithConfig(
    "redis://localhost:6379",
    1*time.Hour,    // Success results kept 1 hour
    24*time.Hour,   // Failure results kept 24 hours
)
```

## Job Submission

### SubmitJob

Submit a job to the default queue.

```go
func (c *Client) SubmitJob(
    name string,
    payload interface{},
    priority Priority,
    description string,
) (string, error)
```

**Parameters:**
- `name` - Job handler name (must be registered in workers)
- `payload` - Job data (must be JSON-serializable)
- `priority` - `PriorityHigh`, `PriorityNormal`, or `PriorityLow`
- `description` - Human-readable job description

**Returns:**
- Job ID (string)
- Error (if any)

**Example:**
```go
jobID, err := c.SubmitJob(
    "process_order",
    map[string]interface{}{
        "order_id": 12345,
        "items": []string{"item1", "item2"},
    },
    client.PriorityNormal,
    "Process customer order #12345",
)
```

### SubmitJobWithRoute

Submit a job to a specific worker pool using routing keys.

```go
func (c *Client) SubmitJobWithRoute(
    name string,
    payload interface{},
    priority Priority,
    routingKey string,
    description string,
) (string, error)
```

**Parameters:**
- `routingKey` - Route to specific worker pool (e.g., "gpu", "email")

**Example:**
```go
// Route to GPU workers
jobID, err := c.SubmitJobWithRoute(
    "train_model",
    map[string]interface{}{"model": "resnet50"},
    client.PriorityHigh,
    "gpu",
    "Train ML model",
)

// Route to email workers
jobID, err := c.SubmitJobWithRoute(
    "send_bulk_email",
    map[string]interface{}{"campaign_id": 456},
    client.PriorityNormal,
    "email",
    "Send marketing campaign",
)
```

### SubmitJobScheduled

Submit a job for future execution.

```go
func (c *Client) SubmitJobScheduled(
    name string,
    payload interface{},
    priority Priority,
    scheduledFor time.Time,
    description string,
) (string, error)
```

**Parameters:**
- `scheduledFor` - When to execute the job

**Example:**
```go
// Schedule for 1 hour from now
executeAt := time.Now().Add(1 * time.Hour)

jobID, err := c.SubmitJobScheduled(
    "send_reminder",
    map[string]interface{}{"user_id": 789},
    client.PriorityNormal,
    executeAt,
    "Send reminder notification",
)
```

### SubmitAndWait

Submit a job and block until result is ready (RPC-style).

```go
func (c *Client) SubmitAndWait(
    ctx context.Context,
    name string,
    payload interface{},
    priority Priority,
    timeout time.Duration,
) (*Result, error)
```

**Parameters:**
- `ctx` - Context for cancellation
- `timeout` - Maximum wait time for job completion

**Returns:**
- Result object with job outcome
- Error (if timeout or job failed)

**Example:**
```go
ctx := context.Background()

result, err := c.SubmitAndWait(
    ctx,
    "calculate_total",
    map[string]interface{}{"items": []int{1, 2, 3, 4, 5}},
    client.PriorityHigh,
    30*time.Second,
)

if err != nil {
    log.Fatal(err)
}

fmt.Printf("Result: %v\n", result.Data)
```

## Job Status & Results

### GetJob

Retrieve job status and metadata.

```go
func (c *Client) GetJob(jobID string) (*Job, error)
```

**Returns:**
```go
type Job struct {
    ID           string
    Name         string
    Payload      interface{}
    Status       JobStatus
    Priority     Priority
    RoutingKey   string
    Description  string
    Attempts     int
    MaxRetries   int
    Error        string
    CreatedAt    time.Time
    ScheduledFor time.Time
    CompletedAt  time.Time
}
```

**Example:**
```go
job, err := c.GetJob("job_abc123")
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Status: %s\n", job.Status)
fmt.Printf("Attempts: %d/%d\n", job.Attempts, job.MaxRetries)
```

### GetResult

Retrieve job result (non-blocking).

```go
func (c *Client) GetResult(ctx context.Context, jobID string) (*Result, error)
```

**Returns:**
```go
type Result struct {
    JobID       string
    Status      string  // "completed" or "failed"
    Data        interface{}
    Error       string
    CompletedAt time.Time
}
```

**Example:**
```go
ctx := context.Background()

result, err := c.GetResult(ctx, "job_abc123")
if err != nil {
    log.Fatal(err)
}

if result.Status == "completed" {
    fmt.Printf("Success: %v\n", result.Data)
} else {
    fmt.Printf("Failed: %s\n", result.Error)
}
```

## Types

### Priority

```go
type Priority string

const (
    PriorityHigh   Priority = "high"
    PriorityNormal Priority = "normal"
    PriorityLow    Priority = "low"
)
```

### JobStatus

```go
type JobStatus string

const (
    StatusPending    JobStatus = "pending"
    StatusProcessing JobStatus = "processing"
    StatusCompleted  JobStatus = "completed"
    StatusFailed     JobStatus = "failed"
    StatusScheduled  JobStatus = "scheduled"
)
```

## Error Handling

```go
// Check for specific errors
jobID, err := c.SubmitJob(...)
if err != nil {
    switch {
    case errors.Is(err, client.ErrRedisConnection):
        log.Println("Redis connection failed")
    case errors.Is(err, client.ErrInvalidPayload):
        log.Println("Payload is not JSON-serializable")
    default:
        log.Printf("Unknown error: %v", err)
    }
}
```

## Best Practices

### 1. Reuse Client Instances

```go
// Good: Single client for application
var globalClient *client.Client

func init() {
    globalClient = client.NewClient("redis://localhost:6379")
}

func submitJob() {
    globalClient.SubmitJob(...)
}
```

### 2. Use Context for Timeouts

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

result, err := c.SubmitAndWait(ctx, ...)
```

### 3. Handle Job Failures

```go
result, err := c.GetResult(ctx, jobID)
if err != nil {
    return err
}

if result.Status == "failed" {
    // Decide: retry, log, alert?
    log.Printf("Job failed: %s", result.Error)
}
```

### 4. Payload Best Practices

```go
// Use structs for type safety
type OrderPayload struct {
    OrderID int      `json:"order_id"`
    Items   []string `json:"items"`
}

payload := OrderPayload{
    OrderID: 12345,
    Items:   []string{"item1", "item2"},
}

jobID, err := c.SubmitJob("process_order", payload, ...)
```

## Related Documentation

- **[REST API](./rest-api)** - HTTP endpoints for job submission
- **[Worker API](./worker-api)** - Creating job handlers
- **[Quick Start](../getting-started/quickstart)** - Usage examples
