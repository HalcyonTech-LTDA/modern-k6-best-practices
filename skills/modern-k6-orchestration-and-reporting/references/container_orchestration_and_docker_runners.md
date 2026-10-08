# Container Orchestration & Docker Runners

Executing Grafana K6 inside containers standardizes performance testing across varied developer environments, eliminating discrepancies between operating systems and local toolchain installations.

---

## 1. Official Container Architecture

Grafana provides official multi-architecture Docker images on Docker Hub (`grafana/k6`).

### Security & User Invariant:
The official `grafana/k6` container runs under an unprivileged user:
- User: `k6`
- UID/GID: `12345:12345`

When mounting host directories into the container to write test reports, ensure the host directory is created before execution or has appropriate permissions:

```bash
# Ensure reports directory exists before running container
mkdir -p k6/reports
chmod 777 k6/reports  # Or assign chown to UID 12345
```

---

## 2. Docker Compose Choreography Pattern

When testing applications orchestrated via Docker Compose, k6 can be integrated as an on-demand testing service that waits for the application and database to become fully healthy:

```yaml
# compose.yaml (or docker-compose.yaml)
services:
  app:
    image: my-app:1.0.0
    environment:
      SPRING_PROFILES_ACTIVE: performance
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 15s
    networks:
      - perf-network

  k6:
    image: grafana/k6:0.54.0
    profiles: ["testing"] # Only runs when explicitly requested
    depends_on:
      app:
        condition: service_healthy
    volumes:
      - ./k6:/scripts:ro
      - ./k6/reports:/reports:rw
    environment:
      TARGET_URL: "http://app:8080" # Direct container DNS resolution!
      K6_ENV: "docker"
      K6_WEB_DASHBOARD: "true"
      K6_WEB_DASHBOARD_EXPORT: "/reports/load-test.html"
    networks:
      - perf-network
    command: ["run", "/scripts/tests/02-load.js"]

networks:
  perf-network:
    driver: bridge
```

### Running via Compose Profile:
```bash
# Spins up app + dependencies, waits for health, runs k6 test, and exits
docker compose run --rm k6
```

---

## 3. Extending K6 with `xk6` Custom Docker Images

If your performance testing scenarios require non-standard protocols (e.g. Kafka, Redis, SQL, MQTT, WebSockets with distributed tracing), k6 allows compiling custom binaries using `xk6`:

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23-alpine AS builder
RUN go install go.k6.io/xk6/cmd/xk6@latest

# Build k6 binary with desired extension
RUN xk6 build v0.54.0 \
  --with github.com/mostafa/xk6-kafka@latest \
  --output /tmp/k6

FROM alpine:3.20
RUN adduser -D -u 12345 k6
COPY --from=builder /tmp/k6 /usr/local/bin/k6
USER 12345
ENTRYPOINT ["k6"]
```
