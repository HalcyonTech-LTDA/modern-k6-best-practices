# Grafana K6 Standard `README.md` Template Blueprint

Use this template as the standard blueprint whenever establishing or maintaining a dedicated `README.md` inside any Grafana K6 performance testing directory.

---

````markdown
# Performance & Load Testing - <Service Name>

This directory contains the automated performance, load, stress, and resilience testing suite for **<Service Name>**, built with [Grafana k6](https://k6.io/).

---

## 1. Directory Structure

```text
<test-root>/
├── README.md                 # Living performance testing runbook and operational guide
├── config/
│   └── environments.js      # Dynamic environment configurations (local, dev, staging, prod)
├── helpers/
│   ├── http-client.js       # Resilient HTTP client wrapper with correlation IDs and timeouts
│   ├── data-generator.js    # Synthetic payload generators (unique identifiers, entities)
│   └── pacing.js            # Realistic think time and pacing utilities with jitter
├── scenarios/
│   ├── <feature-a>-flows.js # Reusable business journey for Feature A (e.g. user creation)
│   └── <feature-b>-flows.js # Reusable business journey for Feature B (e.g. checkout flow)
├── tests/
│   ├── 01-smoke.js          # Baseline sanity verification (1-2 VUs, ~20s)
│   ├── 02-load.js           # Production peak SLA load test (ramping to peak, ~5m)
│   ├── 03-stress.js         # Scaled stress to breaking point (staircase ramping, ~10m)
│   ├── 04-spike.js          # Instant traffic surge and recovery verification (~3m)
│   └── 05-soak.js           # Extended endurance / memory leak test (sustained load, ~4h)
└── reports/                 # Output destination for interactive HTML Web Dashboards & JSON
    └── .gitkeep
```

---

## 2. Target Endpoints & Business Scenarios

The test suite exercises the following application entrypoints:

### <Domain Area A> ([`<TargetControllerA>`](<path/to/controller>))
- `POST /api/v1/<resource>`: Creates a new resource with unique synthetic payload. Validates `201 Created` status and payload integrity.
- `GET /api/v1/<resource>/{id}`: Retrieves resource by identifier. Validates `200 OK` status and identity match.
- `GET /api/v1/<resource>/{non-existent-id}`: Tests resilience against missing records, expecting `404 Not Found`.

### <Domain Area B> ([`<TargetControllerB>`](<path/to/controller>))
- `POST /api/v1/<transaction>`: Executes business transaction for the registered entity. Validates `200 OK` / `201 Created`.
- `GET /api/v1/<transaction>/{id}`: Fetches transaction state by identifier. Validates `200 OK`.

---

## 3. Environment Configuration & Selection

You can switch the target environment dynamically by passing `K6_ENV` (or overriding directly via `TARGET_URL`):

| Environment | Default Base URL | Purpose / Usage |
| :--- | :--- | :--- |
| **`local`** *(default)* | `http://localhost:8080` (or `http://host.docker.internal:8080` in Docker) | Local development verification |
| **`dev`** | `https://dev-api.example.com` | Development cluster integration |
| **`staging`** | `https://staging-api.example.com` | Pre-production validation & capacity verification |
| **`prod`** | `https://api.example.com` | Production verification / dark launch |

### Dynamic Environment Variables

| Variable | Type | Default | Description |
| :--- | :---: | :---: | :--- |
| `K6_ENV` | String | `local` | Target environment preset (`local`, `dev`, `staging`, `prod`) |
| `TARGET_URL` | String | *(derived)* | Explicit base URL override, taking precedence over `K6_ENV` |
| `AUTH_TOKEN` | String | *(none)* | Bearer authentication token for secured endpoints |
| `K6_WEB_DASHBOARD` | Boolean | `false` | Enables the real-time k6 interactive browser dashboard |
| `K6_WEB_DASHBOARD_EXPORT` | String | *(none)* | File path destination for exported interactive HTML dashboard |

---

## 4. Test Types & Workload Profiles Matrix

| Test Profile | Script File | Target Concurrency | Duration | Key SLA Thresholds |
| :--- | :--- | :---: | :---: | :--- |
| **Smoke Test** | `tests/01-smoke.js` | 1 VU | 20s | `http_req_failed: rate==0`<br>`http_req_duration: p(95)<300ms` |
| **Load Test** | `tests/02-load.js` | Ramping to 30 VUs | 5m | `http_req_failed: rate<0.01`<br>`http_req_duration: p(95)<250ms, p(99)<500ms` |
| **Stress Test** | `tests/03-stress.js` | Ramping 10 -> 150 VUs | 10m | `http_req_failed: rate<0.15` (circuit breaker)<br>`http_req_duration: p(95)<1500ms` |
| **Spike Test** | `tests/04-spike.js` | 5 -> 120 VUs in 10s | 3m | `http_req_failed: rate<0.05`<br>`http_req_duration: p(95)<1000ms` |
| **Soak Test** | `tests/05-soak.js` | Constant 25 VUs | 4h | `http_req_failed: rate<0.01`<br>`http_req_duration: p(95)<300ms` |

---

## 5. Execution Runbook

### Option A: Via Taskfile / Make Runner (Recommended)
The project provides automated task runners with an intelligent Docker fallback. If the `k6` CLI is not installed locally, the task runs seamlessly inside the official `grafana/k6` container:

```bash
# Run smoke test against local environment
task k6:smoke

# Run load test targeting staging
task k6:load K6_ENV=staging

# Run stress test targeting staging
task k6:stress K6_ENV=staging

# Run spike test
task k6:spike
```

### Option B: Direct CLI Execution
If you have the `k6` CLI installed locally:

```bash
# 1. Run smoke test targeting local instance
k6 run <test-root>/tests/01-smoke.js

# 2. Run load test with Web Dashboard and HTML export
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=<test-root>/reports/02-load.html \
  k6 run <test-root>/tests/02-load.js

# 3. Run targeting a custom environment and URL
k6 run -e K6_ENV=staging -e TARGET_URL=https://staging.example.com <test-root>/tests/02-load.js
```

### Option C: Standalone Docker Execution
When running without local dependencies:

```bash
docker run --rm -i \
  --network="host" \
  -v "$(pwd)/<test-root>:/scripts" \
  -e K6_WEB_DASHBOARD=true \
  -e K6_WEB_DASHBOARD_EXPORT=/scripts/reports/02-load.html \
  grafana/k6:0.54.0 run /scripts/tests/02-load.js
```

---

## 6. Observability, Web Dashboards & HTML Reports

All automated task runs export an interactive HTML Web Dashboard to `<test-root>/reports/`:
- `<test-root>/reports/01-smoke.html`
- `<test-root>/reports/02-load.html`
- `<test-root>/reports/03-stress.html`
- `<test-root>/reports/04-spike.html`

Open any exported HTML file in your browser to inspect:
- Real-time latency distribution curves (min, med, p90, p95, p99, max)
- Requests per second (RPS) throughput and active VU concurrency
- Threshold pass/fail evaluations and custom domain metrics

---

## 7. Test Data Strategy & Safety Runbook

- **Dynamic Synthetic Data**: Write operations (`POST`, `PUT`) generate unique synthetic identities (names, emails, UUIDs) per iteration to prevent cache contamination and database unique-key collisions.
- **Production Safety**: Stress and Spike profiles must never be executed against production environments without explicit change approval and dark-launch flags.
- **Idempotency & Teardown**: Synthetic records are flagged with a distinctive prefix (e.g. `loadtest-*`) to allow scheduled background pruning.
````
