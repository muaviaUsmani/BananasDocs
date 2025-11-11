---
sidebar_position: 2
---

# Troubleshooting

Common issues and their solutions when running Bananas.

## Redis Connection Issues

### Problem: Cannot Connect to Redis

**Error**: `dial tcp 127.0.0.1:6379: connect: connection refused`

**Solutions**:

1. **Check Redis is running**:
```bash
docker ps | grep redis
# or
redis-cli ping
```

2. **Verify connection URL**:
```bash
# Check environment variable
echo $REDIS_URL

# Should be: redis://localhost:6379
# Or with password: redis://:password@localhost:6379
```

3. **Test connection**:
```bash
redis-cli -h localhost -p 6379 ping
# Should return: PONG
```

4. **Check firewall/network**:
```bash
telnet localhost 6379
# or
nc -zv localhost 6379
```

### Problem: Redis Authentication Failed

**Error**: `NOAUTH Authentication required`

**Solution**:
```bash
# Include password in connection URL
REDIS_URL=redis://:your_password@localhost:6379

# Or use redis-cli with auth
redis-cli -h localhost -p 6379 -a your_password
```

## Job Processing Issues

### Problem: Jobs Stuck in Pending

**Symptoms**:
- Jobs submitted but never processed
- Queue depth keeps growing
- No worker logs showing job processing

**Solutions**:

1. **Check workers are running**:
```bash
docker compose ps worker
# Should show: Up
```

2. **Check worker logs**:
```bash
docker compose logs -f worker
# Look for errors or "Waiting for jobs..."
```

3. **Verify job handler registered**:
```go
// In worker code
registry := worker.NewRegistry()
registry.Register("send_email", handleSendEmail)  // Must match job name
```

4. **Check routing keys match**:
```bash
# Job submitted with routing_key="gpu"
# Worker must have:
ROUTING_KEY=gpu
```

5. **Verify Redis queues**:
```bash
redis-cli LLEN bananas:queue:high
redis-cli LLEN bananas:queue:normal
redis-cli LLEN bananas:queue:processing
```

### Problem: Jobs Timing Out

**Error**: `context deadline exceeded`

**Solutions**:

1. **Increase job timeout**:
```go
// In worker configuration
executor := worker.NewExecutor(registry, 5*time.Minute)  // Increase timeout
```

2. **Respect context in handler**:
```go
func handleLongJob(ctx context.Context, job *Job) error {
    for i := 0; i < 1000; i++ {
        // Check if context cancelled
        select {
        case <-ctx.Done():
            return ctx.Err()  // Stop processing
        default:
            // Continue work
        }

        processItem(i)
    }
    return nil
}
```

3. **Optimize slow operations**:
```go
// Bad: Sequential API calls
for _, user := range users {
    sendEmail(user)  // Slow!
}

// Good: Concurrent processing
var wg sync.WaitGroup
for _, user := range users {
    wg.Add(1)
    go func(u User) {
        defer wg.Done()
        sendEmail(u)
    }(user)
}
wg.Wait()
```

### Problem: Jobs Failing Repeatedly

**Symptoms**:
- Jobs move to dead letter queue
- Error logs show same failure repeatedly
- Max retries exceeded

**Solutions**:

1. **Check error logs**:
```bash
docker compose logs worker | grep ERROR
```

2. **Inspect failed job**:
```bash
# Get job from dead letter queue
redis-cli LRANGE bananas:queue:dead 0 -1

# Get specific job details
curl http://localhost:8080/jobs/{job-id}
```

3. **Common failure causes**:
   - External API down → Wait for recovery or implement circuit breaker
   - Invalid payload → Validate before submission
   - Handler panic → Add error handling and recovery
   - Database timeout → Increase timeout or optimize query

4. **Recover failed jobs**:
```bash
# View dead letter queue
redis-cli LRANGE bananas:queue:dead 0 10

# Move job back to queue (manual recovery)
redis-cli RPOPLPUSH bananas:queue:dead bananas:queue:high
```

## Performance Issues

### Problem: Low Throughput

**Symptoms**:
- Processing < 100 jobs/sec with many workers
- High queue depth
- Workers idle or underutilized

**Solutions**:

1. **Check worker utilization**:
```bash
curl http://localhost:8080/metrics | grep utilization
# Should be > 70%
```

2. **Increase worker concurrency**:
```bash
# Increase from 5 to 20
WORKER_CONCURRENCY=20 docker compose up -d worker
```

3. **Scale workers horizontally**:
```bash
docker compose up -d --scale worker=10
```

4. **Optimize job handlers**:
```go
// Profile slow handlers
import _ "net/http/pprof"

go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()

// Then visit: http://localhost:6060/debug/pprof/
```

5. **Check Redis performance**:
```bash
redis-cli INFO stats | grep ops_per_sec
# Should be > 1000
```

### Problem: High Memory Usage

**Symptoms**:
- Worker/API memory grows continuously
- Out of memory errors
- Container restarts

**Solutions**:

1. **Check memory usage**:
```bash
docker stats
```

2. **Reduce payload sizes**:
```go
// Bad: Large payload in job
client.SubmitJob("process_file", fileContent)  // 10MB!

// Good: Store externally, pass reference
s3URL := uploadToS3(fileContent)
client.SubmitJob("process_file", map[string]string{"url": s3URL})
```

3. **Configure Redis maxmemory**:
```conf
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

4. **Monitor goroutine leaks**:
```go
import "runtime"

// Check goroutine count
log.Printf("Goroutines: %d", runtime.NumGoroutine())
```

## Queue Health Issues

### Problem: Dead Letter Queue Growing

**Symptoms**:
- Dead letter queue has 100+ jobs
- Jobs consistently failing after max retries

**Investigation**:

```bash
# Count dead jobs
redis-cli LLEN bananas:queue:dead

# Sample failed jobs
redis-cli LRANGE bananas:queue:dead 0 5

# Get failure reasons
curl http://localhost:8080/jobs/{dead-job-id}
```

**Solutions**:

1. **Analyze failure patterns**:
   - Same error across jobs? → Fix handler bug
   - External service timeouts? → Increase timeout/add retry logic
   - Invalid payloads? → Add validation before submission

2. **Bulk retry**:
```bash
# Move all dead jobs back to normal queue
while true; do
    redis-cli RPOPLPUSH bananas:queue:dead bananas:queue:normal || break
done
```

3. **Prevent future failures**:
```go
// Add robust error handling
func handleJob(ctx context.Context, job *Job) error {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Panic in job %s: %v", job.ID, r)
        }
    }()

    // Validate input
    if err := validatePayload(job.Payload); err != nil {
        return fmt.Errorf("invalid payload: %w", err)
    }

    // Process with retries for transient errors
    err := processWithRetry(ctx, job)
    return err
}
```

## Monitoring & Alerting

### Key Metrics to Watch

1. **Error Rate**:
```
Alert if: error_rate > 5%
Action: Check logs, investigate common errors
```

2. **Queue Depth**:
```
Alert if: queue_depth > 1000
Action: Scale workers or investigate slow processing
```

3. **Worker Utilization**:
```
Alert if: utilization < 50% or > 95%
Action: Adjust concurrency or scale workers
```

4. **Dead Letter Queue**:
```
Alert if: dead_queue_size > 100
Action: Investigate failure patterns
```

### Health Check Script

```bash
#!/bin/bash
# health_check.sh

# Check API
if ! curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "API unhealthy"
    exit 1
fi

# Check Redis
if ! redis-cli ping > /dev/null 2>&1; then
    echo "Redis unhealthy"
    exit 1
fi

# Check queue depth
QUEUE_DEPTH=$(redis-cli LLEN bananas:queue:normal)
if [ "$QUEUE_DEPTH" -gt 5000 ]; then
    echo "Queue depth critical: $QUEUE_DEPTH"
    exit 1
fi

echo "All checks passed"
```

## Getting Help

If you're still stuck:

1. **Enable debug logging**:
```bash
LOG_LEVEL=debug docker compose up worker
```

2. **Check GitHub Issues**:
   - [Existing issues](https://github.com/muaviaUsmani/Bananas/issues)
   - Search for your error message

3. **Create detailed bug report**:
   - Bananas version
   - Redis version
   - Error logs
   - Steps to reproduce
   - Expected vs actual behavior

4. **Community resources**:
   - [GitHub Discussions](https://github.com/muaviaUsmani/Bananas/discussions)
   - Stack Overflow tag: `bananas-queue`

## Related Documentation

- **[Deployment](./deployment)** - Production setup
- **[Performance](./performance)** - Optimization tips
- **[Architecture](../core-concepts/architecture)** - How it works
