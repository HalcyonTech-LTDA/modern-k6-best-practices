---
name: modern-k6-workloads-and-thresholds
description: Use this skill when modeling workloads and configuring quality gates in Grafana K6: modern scenarios API, executor selection (closed vs open arrival-rate models to prevent Coordinated Omission), workload profiles (smoke, load, stress, spike, soak), and deterministic SLA thresholds.
---

# Modern Grafana K6 Workloads & Thresholds

This skill governs workload orchestration, scenario modeling, executor selection, and deterministic quality gates (thresholds) in Grafana K6.

---

## 1. The Modern `scenarios` Architecture

Legacy k6 tests used flat, root-level options like `export const options = { vus: 10, duration: '30s' };` or `stages: [...]`.

In modern k6, multi-workload tests **must declare explicit scenarios**. Scenarios provide:
- **Independent Workloads**: Run distinct user journeys simultaneously with different traffic profiles.
- **Granular Scheduling**: Start scenarios at specific offsets (`startTime`) or in sequence.
- **Tailored Executors**: Match the mathematical concurrency model to the business reality.
- **Dedicated Environment Tags**: Tag metrics per scenario for isolated reporting.

```javascript
export const options = {
  scenarios: {
    browse_catalog: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },
        { duration: '3m', target: 20 },
        { duration: '1m', target: 0 },
      ],
      exec: 'browseCatalogScenario',
      tags: { journey: 'catalog' },
    },
    checkout_purchases: {
      executor: 'constant-arrival-rate',
      rate: 10, // 10 checkouts per second
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 20,
      maxVUs: 50,
      exec: 'checkoutScenario',
      tags: { journey: 'checkout' },
    },
  },
};
```

---

## 2. Closed Model vs. Open Model (Eliminating Coordinated Omission)

One of the most dangerous traps in performance testing is **Coordinated Omission**:

### The Closed System Trap (`constant-vus`, `ramping-vus`):
When you model load strictly by Virtual Users (VUs), each VU issues a request, waits for the response, and then issues the next request.
- **If the target system slows down**, each VU takes longer to complete a request.
- **As a result, the load generator sends FEWER requests per second**, artificially relieving pressure on the struggling server!
- The test reports stable error rates because the server was never truly subjected to sustained arrival rates.

### The Open System Solution (`constant-arrival-rate`, `ramping-arrival-rate`):
Arrival-rate executors decouple **Request Rate (iterations/sec)** from **Server Response Time**:
- Requests are dispatched at a constant cadence (e.g. 50 requests/sec) regardless of how fast or slow the server responds.
- If the server slows down, k6 dynamically allocates additional VUs from `preAllocatedVUs` up to `maxVUs` to maintain the target arrival rate.
- This models real-world traffic where incoming users do not wait for earlier users' requests to finish.

```javascript
// Open Model: Enforces 50 iterations per second regardless of response time
export const options = {
  scenarios: {
    constant_rate_test: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
      maxVUs: 200,
    },
  },
};
```

---

## 3. The Five Standard Test Profiles

Every production system must maintain five standardized test profiles:

| Test Profile | Purpose | Workload Pattern | Key SLA Focus |
| :--- | :--- | :--- | :--- |
| **1. Smoke Test** | Baseline sanity check; verifies scripts, reachability, and assertion logic. | Minimal load (1-2 VUs, 15-30s). | Zero errors (`http_req_failed: rate==0`), P95 < baseline. |
| **2. Load Test** | Validates performance under normal and expected daily peak traffic. | Ramp-up (1-2m) -> Plateau (5-15m) -> Ramp-down (1-2m). | Meets production SLOs (e.g. P95 < 200ms, failures < 1%). |
| **3. Stress Test** | Identifies system breaking point, saturation limits, and bottleneck resource. | Progressive staircase ramping (e.g. 2x, 3x, 5x peak load). | Graceful degradation, error thresholds, circuit breaker triggers. |
| **4. Spike Test** | Tests sudden traffic surges (e.g. flash sales) and recovery behavior. | Baseline -> Instant spike (10s) -> Sustained surge (1m) -> Immediate drop. | Auto-scaling latency, queue absorption, rapid recovery to normal baseline. |
| **5. Soak Test** | Detects memory leaks, connection pool leaks, and degradation over time. | Moderate sustained load over extended periods (2h to 24h). | Memory stability, garbage collection pauses, steady state response times. |

---

## 4. Deterministic Thresholds: Automated Quality Gates

Thresholds are pass/fail criteria that k6 evaluates automatically against collected metrics. If any threshold is breached, k6 exits with non-zero exit code (`99`), automatically failing the CI/CD pipeline.

### Essential Threshold Patterns:

```javascript
export const options = {
  thresholds: {
    // 1. Failure rate must be less than 1%
    http_req_failed: ['rate<0.01'],

    // 2. Percentile latencies (P90, P95, P99)
    http_req_duration: [
      'p(90)<200', // 90% of requests must complete under 200ms
      'p(95)<300', // 95% of requests must complete under 300ms
      'p(99)<800', // 99% of requests must complete under 800ms
    ],

    // 3. Overall check success rate must exceed 99%
    checks: ['rate>0.99'],

    // 4. Tag-scoped threshold on specific sensitive endpoints
    'http_req_duration{name:POST /orders}': ['p(95)<450'],

    // 5. Custom metric threshold
    'checkout_duration': ['p(95)<1000'],
  },
};
```

### Circuit Breakers: `abortOnFail`
During aggressive stress testing, do not let catastrophic failures continue hammering a crippled server:

```javascript
export const options = {
  thresholds: {
    // Abort test immediately if failure rate exceeds 10%
    http_req_failed: [{ threshold: 'rate<0.10', abortOnFail: true, delayAbort: '10s' }],
  },
};
```

---

## In-Depth References
- [`references/scenarios_and_executors.md`](references/scenarios_and_executors.md) - Deep dive into all 7 k6 executors and scenario scheduling.
- [`references/test_profiles_and_workload_patterns.md`](references/test_profiles_and_workload_patterns.md) - Deep dive into Smoke, Load, Stress, Spike, and Soak configurations.
- [`references/thresholds_and_quality_gates.md`](references/thresholds_and_quality_gates.md) - Deep dive into statistical percentiles, SLA formulas, tag filtering, and CI failure gates.
