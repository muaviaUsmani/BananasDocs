---
sidebar_position: 1
---

# Architecture Overview

Understanding Bananas' architecture will help you make better decisions when designing your job processing systems.

## System Design Principles

Bananas is built on four core principles:

1. **Simplicity** - Easy to understand, deploy, and maintain
2. **High Performance** - 1,000+ jobs/second with minimal latency
3. **Reliability** - No job loss through atomic Redis operations
4. **Horizontal Scalability** - Add more workers as load increases

## Core Components

### 1. Client SDK

The Go client library provides the interface for submitting jobs:

```go
client := client.NewClient("redis://localhost:6379")
jobID, err := client.SubmitJob("send_email", payload, PriorityHigh, "desc")
```

**Key Features:**
- Job submission with priority and routing
- RPC-style blocking with `SubmitAndWait()`
- Result retrieval without polling (uses Redis pub/sub)
- Scheduled job support

### 2. Job Queue (Redis-Based)

Redis serves as the message broker and data store, using multiple data structures:

**Priority Queues:**
- `bananas:queue:high` - List (FIFO)
- `bananas:queue:normal` - List (FIFO)
- `bananas:queue:low` - List (FIFO)

**Processing Queue:**
- `bananas:queue:processing` - List of in-flight jobs

**Scheduled Set:**
- `bananas:queue:scheduled` - Sorted set (ordered by execution time)

**Dead Letter Queue:**
- `bananas:queue:dead` - Failed jobs after max retries

**Job Data:**
- `bananas:job:{id}` - Hash containing job details

### 3. Worker Pool

Workers are goroutine-based concurrent processors:

```
┌─────────────────────────────┐
│      Worker Instance        │
├─────────────────────────────┤
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐   │
│  │ G │ │ G │ │ G │ │ G │   │  G = Goroutine
│  │ 1 │ │ 2 │ │ 3 │ │...│   │
│  └───┘ └───┘ └───┘ └───┘   │
│         ↓      ↓      ↓     │
│      Handler Registry       │
└─────────────────────────────┘
         ↓
    Redis Queue
```

**Worker Modes:**

1. **Thin Mode** - Only high priority jobs
2. **Default Mode** - High and normal priority
3. **Specialized Mode** - High, normal, low priority
4. **Job-Specialized** - Specific routing keys only
5. **Scheduler-Only** - No job processing

Configure via environment:
```bash
WORKER_MODE=specialized
WORKER_CONCURRENCY=20
ROUTING_KEY=gpu
```

### 4. Executor

The executor handles individual job execution:

```go
type Executor struct {
    registry   *Registry      // Job handlers
    timeout    time.Duration  // Per-job timeout
    resultBackend *ResultBackend
}

func (e *Executor) Execute(ctx context.Context, job *Job) error {
    handler, ok := e.registry.Get(job.Name)
    if !ok {
        return fmt.Errorf("handler not found: %s", job.Name)
    }

    // Execute with timeout
    ctx, cancel := context.WithTimeout(ctx, e.timeout)
    defer cancel()

    result := handler(ctx, job)

    // Store result
    e.resultBackend.Store(job.ID, result)

    return nil
}
```

**Responsibilities:**
- Look up job handler from registry
- Enforce per-job timeouts
- Catch panics and log stack traces
- Store results in result backend
- Trigger retry on failure

### 5. Scheduler

Two scheduling components work together:

**Periodic Task Scheduler (Cron):**
```go
scheduler.AddSchedule(&Schedule{
    Name:       "daily_report",
    Cron:       "0 2 * * *",  // 2 AM daily
    Job:        "generate_report",
    Payload:    map[string]interface{}{"type": "daily"},
    Priority:   PriorityNormal,
    Timezone:   "America/New_York",
})
```

**Retry Scheduler:**
- Runs every second
- Checks `bananas:queue:scheduled` sorted set
- Moves jobs with `scheduled_for <= now()` to appropriate priority queues
- Uses distributed locking to prevent duplicate processing

### 6. Result Backend

Stores job outcomes with configurable TTL:

```go
type Result struct {
    JobID      string
    Status     string  // "completed" or "failed"
    Data       interface{}
    Error      string
    CompletedAt time.Time
}
```

**Storage:**
- Success results: 1 hour TTL (configurable)
- Failure results: 24 hours TTL (configurable)
- Uses Redis hashes: `bananas:result:{job_id}`

**Notifications:**
- Publishes to Redis channel on completion
- Enables `SubmitAndWait()` without polling

## Critical Design Decisions

### Blocking Dequeue vs Polling

**Decision: Blocking Dequeue**

```go
// Uses Redis BRPOPLPUSH - blocks until job available
job, err := redis.BRPopLPush("bananas:queue:high", "bananas:queue:processing", 5*time.Second)
```

**Benefits:**
- Instant processing (no polling delay)
- Zero CPU usage when idle
- Reduced Redis load
- Natural back-pressure mechanism

**Alternative (NOT used):**
```go
// Polling approach - wastes resources
for {
    job := redis.RPop("queue")
    if job == nil {
        time.Sleep(100 * time.Millisecond)  // Busy-wait
        continue
    }
    process(job)
}
```

### Exponential Backoff for Retries

**Decision: 2^attempt seconds**

```go
retryDelay := time.Duration(math.Pow(2, float64(job.Attempts))) * time.Second
job.ScheduledFor = time.Now().Add(retryDelay)
```

**Retry Timeline:**
- 1st retry: 2 seconds (2^1)
- 2nd retry: 4 seconds (2^2)
- 3rd retry: 8 seconds (2^3)
- 4th retry: 16 seconds (2^4)
- 5th retry: 32 seconds (2^5)

**Why Exponential?**
- Gives failing dependencies time to recover
- Prevents thundering herd problems
- Reduces load on struggling systems
- Natural circuit breaker pattern

**Alternative (NOT used):**
Linear retry (1s, 2s, 3s...) doesn't give sufficient recovery time for overwhelmed services.

### Routing Keys for Resource Isolation

**Decision: Route jobs to specialized worker pools**

```go
// GPU-intensive jobs
client.SubmitJobWithRoute("train_model", payload, PriorityHigh, "gpu", "desc")

// Email jobs
client.SubmitJobWithRoute("send_bulk_email", payload, PriorityNormal, "email", "desc")
```

**Queue Structure:**
```
bananas:queue:gpu:high
bananas:queue:gpu:normal
bananas:queue:email:high
bananas:queue:email:normal
```

**Benefits:**
- Resource isolation (GPU workers don't process email jobs)
- Independent scaling (scale GPU and email workers separately)
- Priority within routing groups
- Prevents resource contention

### Goroutine-Based Workers

**Decision: Lightweight goroutines over threads**

**Comparison:**
| Feature | Goroutines | OS Threads |
|---------|-----------|-----------|
| Stack Size | 2 KB | 1-2 MB |
| Creation Time | ~1 μs | ~3 ms |
| Context Switch | ~0.2 μs | ~1-2 μs |
| Max Concurrent | 10,000+ | ~1,000 |

**Code:**
```go
for i := 0; i < concurrency; i++ {
    go func() {
        for {
            job := dequeueJob()
            processJob(job)
        }
    }()
}
```

**Why Goroutines?**
- Can run 10,000+ concurrent workers on single machine
- Minimal memory footprint
- Fast context switching
- Go scheduler handles load balancing
- Perfect for I/O-bound tasks (API calls, database queries)

## Data Flow

### Job Submission Flow

```
Client → Redis → Worker → Handler → Result Backend
  │        │        │        │            │
  ├─1. Submit job  │        │            │
  │        ├─2. Store in queue           │
  │        │        ├─3. Dequeue (BRPOPLPUSH)
  │        │        │        ├─4. Execute
  │        │        │        └─5. Return result
  │        │        │                    ├─6. Store result
  └────────┴────────┴────────────────────┴─7. Publish completion
```

### Priority Processing Flow

```
┌─────────────────────────────────────┐
│          Worker Pool                │
└─────────────────────────────────────┘
              ↓ Dequeue
    ┌──────────────────────┐
    │    High Priority     │ ← Checked first
    ├──────────────────────┤
    │   Normal Priority    │ ← If high empty
    ├──────────────────────┤
    │    Low Priority      │ ← If normal empty
    └──────────────────────┘
```

Workers check queues in strict priority order, ensuring high-priority jobs never wait behind lower-priority ones.

## Performance Characteristics

### Throughput
- **8,000-9,000 jobs/sec** submission rate
- **1,650 jobs/sec** end-to-end processing (20 workers)
- **12,000+ ops/sec** with concurrent clients

### Latency
- **< 3ms p99** job submission
- **< 100ms p99** end-to-end (including execution)
- **< 1ms** queue operations (Redis)

### Scalability
- **Linear horizontal scaling** - 2x workers ≈ 2x throughput
- **10,000+ concurrent workers** on single machine
- **No application-level locks** - scales with Redis

## Thread Safety

Bananas achieves thread safety without application-level locks:

**Redis Atomic Operations:**
- `BRPOPLPUSH` - Atomic dequeue and move to processing
- `SET NX` - Atomic scheduler locking
- `HINCRBY` - Atomic counter updates
- Pipelining for batched operations

**Stateless Workers:**
- No shared memory between workers
- Each goroutine has isolated stack
- Communication only via Redis
- Can be distributed across machines

## Next Steps

- **[Job Lifecycle](./job-lifecycle)** - How jobs progress through the system
- **[Queues and Priorities](./queues)** - Deep dive into queue management
- **[Workers](./workers)** - Worker modes and configuration
- **[API Reference](../category/api-reference)** - Complete API docs
