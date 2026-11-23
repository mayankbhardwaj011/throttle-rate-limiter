# Throttle — Distributed Rate Limiter

A distributed rate limiter built with Spring Boot and Redis. I built this to get a solid understanding of how rate limiting actually works under the hood — from the algorithm choices to making it work across multiple service instances using atomic Redis operations.

---

## What it does

At its core, this service exposes a REST API that tells you whether a given key (user ID, API key, IP address — whatever you want) has exceeded its rate limit. The state lives in Redis, so it works correctly even when you're running multiple instances of your app.

You can pick from five different algorithms depending on your use case:

- **Token Bucket** — the classic. Tokens refill at a steady rate, and burst traffic is absorbed up to the bucket capacity. Good general-purpose choice.
- **Sliding Window** — counts requests in a rolling time window, so there are no sharp resets at the boundary like with fixed windows.
- **Fixed Window** — simple counter per time window. Cheap and predictable, but can allow 2x the limit right at a window boundary.
- **Leaky Bucket** — smooths out traffic by draining at a constant rate. Useful when you want to shape traffic, not just cap it.
- **Composite** — chain multiple algorithms together. For example: token bucket for burst tolerance + sliding window for sustained rate enforcement.

---

## Tech stack

- **Java 21** + **Spring Boot 4.1.2**
- **Redis** for distributed state (Lua scripts for atomic operations)
- **Testcontainers** for integration tests (spins up a real Redis)
- **React + TypeScript** for the web dashboard (`examples/web-dashboard/`)
- **Docker** + **Kubernetes** manifests for deployment

---

## Getting started

### Prerequisites

- Java 21
- Docker (for Redis and integration tests)

### Run locally

```bash
# Start Redis
docker compose up -d redis

# Start the app
./mvnw spring-boot:run
```

The service starts on `http://localhost:8080`.  
Swagger UI is at `http://localhost:8080/swagger-ui/index.html`.

### Run with Docker Compose (full stack)

```bash
docker compose up
```

This starts the rate limiter + Redis together.

---

## Core API

### Check a rate limit

```http
POST /api/rate-limit/check
Content-Type: application/json

{
  "key": "user:123",
  "algorithm": "TOKEN_BUCKET"
}
```

**Response when allowed:**
```json
{
  "allowed": true,
  "remainingTokens": 49,
  "resetAfterMs": 1000
}
```

**Response when limited (HTTP 429):**
```json
{
  "allowed": false,
  "remainingTokens": 0,
  "resetAfterMs": 850
}
```

### Configure per-key limits

```http
POST /api/config/keys
Content-Type: application/json

{
  "key": "api-key:premium-user",
  "algorithm": "TOKEN_BUCKET",
  "capacity": 1000,
  "refillRate": 100
}
```

You can also configure pattern-based rules (e.g. `user:*`) so you don't have to register every key individually.

### Admin endpoints

```http
GET  /api/admin/keys              # List all active keys
DELETE /api/admin/keys/{key}      # Remove a key's state
GET  /api/admin/stats             # Overall system stats
```

Admin endpoints are protected with API key authentication — set `X-Admin-Key` in the request header.

---

## Configuration

Main config lives in `src/main/resources/application.properties`:

```properties
# Redis
spring.data.redis.host=localhost
spring.data.redis.port=6379

# Default rate limit (applied when no key-specific config exists)
rate-limiter.default.algorithm=TOKEN_BUCKET
rate-limiter.default.capacity=100
rate-limiter.default.refill-rate=10

# Admin
rate-limiter.admin.api-key=your-secret-key
```

---

## Web dashboard

There's a React dashboard in `examples/web-dashboard/` for visualizing what's happening in real time.

```bash
cd examples/web-dashboard
npm install
npm run dev
# http://localhost:5173
```

It shows live request metrics, lets you switch algorithms, run load tests, and manage per-key configuration — all backed by the actual REST API.

---

## Running tests

```bash
# All tests (requires Docker for Testcontainers)
./mvnw test

# Specific test class
./mvnw test -Dtest=TokenBucketTest
./mvnw test -Dtest=RateLimiterServiceTest

# Skip integration tests
./mvnw test -DskipITs
```

There are 500+ tests covering unit, integration, concurrency, and load scenarios.

---

## Deployment

### Kubernetes

Manifests are in the `k8s/` directory:

```bash
kubectl apply -f k8s/base/
```

### Docker image

```bash
# Build
./mvnw clean package
docker build -t throttle-rate-limiter .

# Run
docker run -p 8080:8080 -e SPRING_REDIS_HOST=your-redis throttle-rate-limiter
```

---

## Project structure

```
src/
  main/java/dev/bnacar/distributedratelimiter/
    ratelimit/        # Algorithm implementations
    config/           # Rate limit configuration & per-key rules
    admin/            # Admin management endpoints
    monitoring/       # Metrics & health
    security/         # API key auth, IP filtering
    observability/    # Structured logging, tracing
examples/
  web-dashboard/      # React monitoring dashboard
k8s/                  # Kubernetes manifests
docs/                 # API reference, ADRs, examples
```
