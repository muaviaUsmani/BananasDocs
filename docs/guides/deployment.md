---
sidebar_position: 1
---

# Production Deployment

Complete guide for deploying Bananas to production environments.

## Overview

Bananas can be deployed using:
- **Docker Compose** - Simple deployments (< 10,000 jobs/hour)
- **Kubernetes** - Large-scale production (> 10,000 jobs/hour)
- **Systemd** - Bare metal/VM deployments

## Docker Compose Deployment

### Quick Production Setup

```bash
# Clone repository
git clone https://github.com/muaviaUsmani/Bananas.git
cd Bananas

# Build production images
make prod-build

# Start services
docker compose up -d

# Verify
docker compose ps
curl http://localhost:8080/health
```

### Production docker-compose.yml

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

  api:
    image: bananas-api:latest
    environment:
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
      - API_PORT=8080
      - LOG_LEVEL=info
    ports:
      - "8080:8080"
    depends_on:
      redis:
        condition: service_healthy
    restart: always

  worker:
    image: bananas-worker:latest
    environment:
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
      - WORKER_CONCURRENCY=20
      - WORKER_MODE=default
      - LOG_LEVEL=info
    depends_on:
      redis:
        condition: service_healthy
    deploy:
      replicas: 3
    restart: always

  scheduler:
    image: bananas-scheduler:latest
    environment:
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
      - LOG_LEVEL=info
    depends_on:
      redis:
        condition: service_healthy
    restart: always

volumes:
  redis-data:
```

### Environment Variables (.env.prod)

```bash
# Redis
REDIS_PASSWORD=your_strong_password_here

# API
API_PORT=8080

# Workers
WORKER_CONCURRENCY=20
WORKER_MODE=default

# Result TTL
RESULT_TTL_SUCCESS=3600
RESULT_TTL_FAILURE=86400

# Logging
LOG_LEVEL=info
```

### Scaling Workers

```bash
# Scale to 10 workers
docker compose up -d --scale worker=10

# Or use make command
make scale-workers N=10
```

## Kubernetes Deployment

### Prerequisites

- Kubernetes cluster (1.19+)
- kubectl configured
- Helm 3+ (optional)

### Redis StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
          - redis-server
          - "--requirepass"
          - "$(REDIS_PASSWORD)"
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### Worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bananas-worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: bananas-worker
  template:
    metadata:
      labels:
        app: bananas-worker
    spec:
      containers:
      - name: worker
        image: bananas-worker:latest
        env:
        - name: REDIS_URL
          value: "redis://:$(REDIS_PASSWORD)@redis:6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        - name: WORKER_CONCURRENCY
          value: "20"
        - name: WORKER_MODE
          value: "default"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bananas-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bananas-worker
  minReplicas: 5
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: queue_depth
      target:
        type: AverageValue
        averageValue: "100"
```

### Deploy to Kubernetes

```bash
# Create namespace
kubectl create namespace bananas

# Create secrets
kubectl create secret generic redis-secret \
  --from-literal=password='your_password' \
  -n bananas

# Apply manifests
kubectl apply -f k8s/ -n bananas

# Verify
kubectl get pods -n bananas
kubectl logs -f deployment/bananas-worker -n bananas
```

## Redis Configuration

### Production redis.conf

```conf
# Persistence
save 900 1
save 300 10
save 60 10000

# AOF
appendonly yes
appendfsync everysec

# Memory
maxmemory 2gb
maxmemory-policy allkeys-lru

# Security
requirepass your_strong_password_here
rename-command FLUSHDB ""
rename-command FLUSHALL ""

# Networking
bind 0.0.0.0
protected-mode yes

# Performance
tcp-backlog 511
timeout 300
```

### Redis Cluster (High Availability)

For > 100,000 jobs/hour:

```bash
# 3-node Redis Cluster
redis-cli --cluster create \
  redis1:6379 redis2:6379 redis3:6379 \
  --cluster-replicas 0

# Update connection URL
REDIS_URL=redis://redis1:6379,redis2:6379,redis3:6379
```

## Monitoring & Metrics

### Prometheus Integration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'bananas-api'
    static_configs:
      - targets: ['api:8080']
    metrics_path: '/metrics'

  - job_name: 'bananas-workers'
    static_configs:
      - targets: ['worker:9090']
```

### Key Metrics to Monitor

- **Queue Depth**: `bananas_queue_depth{priority="high"}`
- **Job Throughput**: `bananas_jobs_processed_total`
- **Error Rate**: `bananas_jobs_failed_total`
- **Worker Utilization**: `bananas_worker_utilization`
- **Redis Memory**: `redis_memory_used_bytes`

### Grafana Dashboard

Import dashboard ID: `bananas-overview-12345`

Key panels:
- Jobs/sec (submission & processing)
- Queue depth by priority
- Worker utilization
- Error rate
- P95/P99 latencies

## Security

### TLS for Redis

```yaml
# Enable TLS
redis:
  args:
    - --tls-port 6380
    - --tls-cert-file /certs/redis.crt
    - --tls-key-file /certs/redis.key
    - --tls-ca-cert-file /certs/ca.crt
```

### Network Policies (Kubernetes)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: bananas-network-policy
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: bananas-worker
    - podSelector:
        matchLabels:
          app: bananas-api
    ports:
    - protocol: TCP
      port: 6379
```

## Backup & Recovery

### Redis Backup Strategy

```bash
# RDB snapshot every hour
0 * * * * docker exec redis redis-cli BGSAVE

# Copy to S3
0 * * * * docker cp redis:/data/dump.rdb /backup/ && \
          aws s3 cp /backup/dump.rdb s3://backups/redis/$(date +%Y%m%d-%H%M).rdb
```

### Restore from Backup

```bash
# Stop Redis
docker compose stop redis

# Restore RDB file
docker cp backup.rdb redis:/data/dump.rdb

# Start Redis
docker compose start redis
```

## Capacity Planning

### Small Scale (< 1,000 jobs/hour)
- **Redis**: 1 instance (2 CPU, 4GB RAM)
- **Workers**: 2-3 instances
- **API**: 1 instance

### Medium Scale (1,000-10,000 jobs/hour)
- **Redis**: 1 master + 1 replica (4 CPU, 8GB RAM)
- **Workers**: 5-10 instances
- **API**: 2 instances (load balanced)

### Large Scale (> 100,000 jobs/hour)
- **Redis**: Cluster (3-6 nodes, 8 CPU, 16GB RAM each)
- **Workers**: 20-50+ instances
- **API**: 5+ instances (load balanced)

## Health Checks

### API Health Endpoint

```bash
curl http://api:8080/health
# Returns: {"status":"healthy","redis":"connected"}
```

### Liveness/Readiness Probes (K8s)

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Related Documentation

- **[Troubleshooting](./troubleshooting)** - Common issues
- **[Performance](./performance)** - Optimization tips
- **[Architecture](../core-concepts/architecture)** - System design
