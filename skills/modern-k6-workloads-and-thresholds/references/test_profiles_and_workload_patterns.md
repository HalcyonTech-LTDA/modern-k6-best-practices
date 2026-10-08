# Test Profiles & Workload Patterns

An enterprise performance testing strategy requires multiple distinct workload profiles. Running only a single load test fails to answer crucial operational questions: *Where is the breaking point? Does the system recover from surges? Are there memory leaks over time?*

---

## 1. Profile 1: Smoke Test (Baseline Sanity)

The Smoke Test verifies that test scripts, endpoints, environment configs, and assertion checks are functioning properly with minimal system load.

### Characteristics:
- **Concurrency**: 1 to 2 VUs
- **Duration**: 15 to 30 seconds
- **Purpose**: Fast feedback in local development and PR validation pipelines before launching heavy tests.

```javascript
// tests/01-smoke.js
export const options = {
  vus: 1,
  duration: '20s',
  thresholds: {
    http_req_failed: ['rate==0'], // Zero failures tolerated in smoke
    http_req_duration: ['p(95)<300'],
    checks: ['rate==1.0'], // 100% checks must pass
  },
};
```

---

## 2. Profile 2: Load Test (Peak SLA Verification)

The Load Test assesses system performance under normal and anticipated peak concurrent user volumes.

### Characteristics:
- **Stages**: Ramp-up (warm-up cache & connections) -> Sustained Peak -> Ramp-down.
- **Duration**: 5 to 15 minutes.
- **Purpose**: Validate compliance with production Service Level Objectives (SLOs).

```javascript
// tests/02-load.js
export const options = {
  scenarios: {
    peak_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 30 },  // Ramp-up to peak
        { duration: '5m', target: 30 },  // Plateau: sustained peak
        { duration: '1m', target: 0 },   // Ramp-down
      ],
      gracefulRampDown: '30s',
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],    // < 1% error rate
    http_req_duration: ['p(95)<250', 'p(99)<500'],
  },
};
```

---

## 3. Profile 3: Stress Test (Breaking Point Discovery)

The Stress Test pushes the system beyond normal capacity to determine the breaking point, understand degradation modes, and evaluate recovery.

### Characteristics:
- **Stages**: Staircase progression (e.g. 10 -> 50 -> 100 -> 150 VUs).
- **Duration**: 10 to 20 minutes.
- **Purpose**: Find bottlenecks (database connection pool limits, thread pool saturation, CPU throttling).

```javascript
// tests/03-stress.js
export const options = {
  scenarios: {
    stress_scaling: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },  // Baseline
        { duration: '2m', target: 50 },  // Normal peak
        { duration: '2m', target: 100 }, // 2x peak
        { duration: '2m', target: 150 }, // 3x peak: stress zone
        { duration: '2m', target: 200 }, // Breaking point exploration
        { duration: '2m', target: 0 },   // Recovery observation
      ],
    },
  },
  thresholds: {
    // Graceful circuit breaker: abort if failure rate reaches 20%
    http_req_failed: [{ threshold: 'rate<0.20', abortOnFail: true, delayAbort: '15s' }],
    http_req_duration: ['p(95)<1500'],
  },
};
```

---

## 4. Profile 4: Spike Test (Surge Traffic & Recovery)

The Spike Test evaluates system resilience when subjected to an abrupt, massive surge in traffic, followed by an immediate return to normal levels.

### Characteristics:
- **Stages**: Baseline -> Instant Surge (e.g. in 10s) -> Short Peak -> Instant Drop -> Baseline Observation.
- **Duration**: 2 to 5 minutes.
- **Purpose**: Verify auto-scaling triggers, queue backpressure, and whether the service recovers without manual restarts.

```javascript
// tests/04-spike.js
export const options = {
  scenarios: {
    flash_sale_spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 5 },   // Normal low baseline
        { duration: '10s', target: 120 }, // Instant violent spike!
        { duration: '1m', target: 120 },  // Peak hold
        { duration: '15s', target: 5 },   // Sudden collapse
        { duration: '1m', target: 5 },    // Observation: does latency recover?
        { duration: '15s', target: 0 },   // Ramp-down
      ],
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.05'], // Max 5% error during spike
    http_req_duration: ['p(95)<1000'],
  },
};
```

---

## 5. Profile 5: Soak / Endurance Test (Leak Detection)

The Soak Test maintains moderate to high load over an extended duration to detect slow-developing faults.

### Characteristics:
- **Duration**: 2 to 24 hours.
- **Purpose**: Surface memory leaks, JVM Metaspace exhaustion, unclosed database/HTTP connections, and disk log filling.

```javascript
// tests/05-soak.js
export const options = {
  scenarios: {
    soak_endurance: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 25 },   // Warm-up
        { duration: '4h', target: 25 },   // Extended steady state
        { duration: '5m', target: 0 },    // Cooldown
      ],
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<300'],
  },
};
```
