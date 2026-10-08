---
name: modern-k6-test-craftsmanship
description: Use this skill when authoring, structuring, or refactoring Grafana K6 test scripts: modern ES6+ modular architecture, standardized HTTP client abstraction, URL metric tag grouping to prevent high cardinality, custom metrics (Trend, Rate, Counter, Gauge), defensive response parsing, non-halting checks, and memory-efficient SharedArray parameterization.
---

# Modern Grafana K6 Test Craftsmanship

This skill governs the code craftsmanship, architectural patterns, and execution mechanics of writing production-grade Grafana K6 performance testing scripts.

---

## 1. The Four K6 Test Execution Lifecycles

Every k6 test script executes within distinct runtime lifecycle stages. Understanding these stages is essential for resource efficiency, avoiding memory leaks, and deterministic test execution:

```text
 ┌─────────────────────────────────────────────────────────────┐
 │ 1. Init Stage (runs ONCE per VU on script compilation)      │
 │    - Import modules, read files with open(), SharedArray    │
 │    - Declare custom metrics (Trend, Counter, Rate, Gauge)   │
 │    - Export options (scenarios, thresholds, stages)         │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 2. Setup Stage (runs ONCE globally on VU 0)                 │
 │    - Provision remote test accounts, obtain auth tokens     │
 │    - Returns data object passed to all VUs                  │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 3. VU Default Stage (runs REPEATEDLY per virtual user)       │
 │    - Executes business journeys, HTTP requests, checks      │
 │    - Simulates think times (sleep), increments metrics      │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 4. Teardown Stage (runs ONCE globally on VU 0 after test)   │
 │    - Cleans up seeded data, closes remote test sessions     │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 5. handleSummary Stage (runs ONCE after test completion)    │
 │    - Processes raw metrics, exports custom HTML/JUnit/JSON  │
 └─────────────────────────────────────────────────────────────┘
```

### Invariant Rules across Lifecycles:
- **Never issue HTTP requests in the Init stage**: HTTP requests are only permitted in `setup()`, default VU execution, or `teardown()`.
- **Never load dynamic files per iteration**: `open()` can only be invoked in the init stage.
- **Never store VU state in global variables**: The default function runs concurrently across virtual users; VU-specific state must remain scoped to the function iteration.

---

## 2. Recommended Directory Architecture

Do not write monolithic, all-in-one test scripts. Maintain a clean, modular structure:

```text
k6/
├── README.md                 # Mandatory living runbook documentation
├── config/
│   └── environments.js      # Base URLs, auth secrets, timeouts per environment
├── helpers/
│   ├── http-client.js       # Resilient HTTP client wrapper with correlation IDs
│   ├── data-generator.js    # Realistic synthetic payload generators
│   └── metrics.js           # Centralized custom metrics declarations
├── data/
│   └── test-users.json      # Large data fixtures loaded via SharedArray
├── scenarios/
│   ├── auth-flows.js        # Reusable user authentication journey
│   ├── catalog-flows.js     # Search and browsing journey
│   └── checkout-flows.js    # Order submission journey
├── tests/
│   ├── 01-smoke.js          # Baseline sanity verification
│   ├── 02-load.js           # Production peak SLA load test
│   ├── 03-stress.js         # Scaled stress to breaking point
│   ├── 04-spike.js          # Instant traffic surge and recovery
│   └── 05-soak.js           # Extended endurance / memory leak test
└── reports/                 # Output directory for HTML and JSON reports
```

---

## 3. High-Cardinality Metric Explosion Prevention

### The Problem:
k6 tracks every unique URL path as a separate time-series metric dimension by default. If your script issues requests with dynamic IDs:
```javascript
// ❌ ANTI-PATTERN: Creates thousands of metrics timeseries!
http.get(`https://api.example.com/api/v1/customers/${customerId}`);
```
This causes an **unbounded high-cardinality metric explosion**, crashing the load generator machine and causing memory saturation in metric backends (Prometheus, InfluxDB, Grafana Mimir).

### The Solution:
Always group dynamic URLs using either the `http.url` template tag or explicit `tags: { name: '...' }`:

```javascript
// ✅ BEST PRACTICE Option A: Tagging via name property
http.get(`${baseUrl}/api/v1/customers/${customerId}`, {
  tags: { name: 'GET /api/v1/customers/:id' },
});

// ✅ BEST PRACTICE Option B: Using http.url template literal
http.get(http.url`${baseUrl}/api/v1/customers/${customerId}`);
```

---

## 4. Standardized Resilient HTTP Client Wrapper

Encapsulate HTTP operations within a centralized wrapper class that uniformly injects timeouts, common headers, distributed tracing IDs (`X-Correlation-ID`), and automatic tag grouping:

```javascript
// helpers/http-client.js
import http from 'k6/http';

export class ResilientHttpClient {
  constructor(config) {
    this.baseUrl = config.baseUrl;
    this.timeout = config.timeout || '10s';
    this.defaultHeaders = config.headers || {};
  }

  request(method, endpoint, body = null, options = {}) {
    const url = `${this.baseUrl}${endpoint}`;
    
    // Generate unique correlation ID for distributed tracing
    const correlationId = `k6-${Date.now()}-${Math.floor(Math.random() * 1000000)}`;

    const params = {
      timeout: this.timeout,
      headers: Object.assign(
        {
          'Content-Type': 'application/json',
          'User-Agent': 'k6-load-testing/1.0',
          'X-Correlation-ID': correlationId,
        },
        this.defaultHeaders,
        options.headers
      ),
      tags: Object.assign(
        {
          name: options.name || `${method} ${endpoint}`,
        },
        options.tags
      ),
    };

    const payload = body && typeof body === 'object' ? JSON.stringify(body) : body;
    return http.request(method, url, payload, params);
  }

  get(endpoint, options = {}) {
    return this.request('GET', endpoint, null, options);
  }

  post(endpoint, body, options = {}) {
    return this.request('POST', endpoint, body, options);
  }

  put(endpoint, body, options = {}) {
    return this.request('PUT', endpoint, body, options);
  }

  del(endpoint, options = {}) {
    return this.request('DELETE', endpoint, null, options);
  }
}
```

---

## 5. Defensive Response Parsing & Non-Halting Checks

In k6, any unhandled JavaScript exception in a virtual user immediately aborts that VU iteration, corrupting aggregate metrics.

### Golden Rules:
1. **Never call `JSON.parse(res.body)` blindly**: An API under heavy load will occasionally return HTML error pages (`502 Bad Gateway`, `504 Gateway Timeout`, `500 Internal Error`). Calling `JSON.parse()` on HTML triggers a fatal syntax error.
2. **Always verify status code or content before parsing**:
   ```javascript
   // ✅ SAFE PARSING PATTERN
   export function parseJsonSafely(res) {
     if (!res || !res.body || res.status < 200 || res.status >= 300) {
       return null;
     }
     try {
       return JSON.parse(res.body);
     } catch (err) {
       console.error(`Failed to parse response body from ${res.url}: ${err.message}`);
       return null;
     }
   }
   ```
3. **Use `check()` for assertions**: Checks do not throw errors; they evaluate booleans and increment the built-in `checks` metric. Pair checks with thresholds to assert quality gates.

---

## 6. Custom Metrics for Domain Precision

Built-in HTTP metrics capture overall traffic, but domain SLOs require custom metrics:

```javascript
// helpers/metrics.js
import { Trend, Rate, Counter, Gauge } from 'k6/metrics';

// Trend: tracks timing percentiles (e.g. checkout latency excluding token fetch)
export const checkoutDuration = new Trend('checkout_duration', true);

// Rate: tracks fraction of operations that succeeded
export const orderSuccessRate = new Rate('order_success_rate');

// Counter: tracks cumulative counts of business events
export const inventoryDepletionCount = new Counter('inventory_depletions');

// Gauge: tracks point-in-time values (note: k6 Gauge uses .add(val), not .set(val))
export const activeCartSize = new Gauge('active_cart_size');
```

Record metric points in your user journeys:
```javascript
const start = Date.now();
const res = client.post('/api/v1/orders', payload, { name: 'POST /orders' });
const duration = Date.now() - start;

checkoutDuration.add(duration);
orderSuccessRate.add(res.status === 201);
if (res.status === 409) {
  inventoryDepletionCount.add(1);
}
// Point-in-time gauge recording (always use .add(val), never .set(val))
activeCartSize.add(payload.items ? payload.items.length : 1);
```

---

## 7. Memory-Efficient Parameterization with `SharedArray`

When parameterizing tests with external data (e.g., 50,000 user credentials or account numbers), reading files with `JSON.parse(open('users.json'))` in global scope creates a **duplicate memory copy for each VU**. With 500 VUs and a 10MB JSON file, that consumes 5GB of RAM.

**Always use `SharedArray` from `k6/data`**:
```javascript
import { SharedArray } from 'k6/data';

export const testUsers = new SharedArray('registered-users', function () {
  const data = JSON.parse(open('../data/test-users.json'));
  return data; // Shared in memory across all VUs as read-only!
});

export default function () {
  // Deterministic or pseudo-random selection per VU
  const user = testUsers[(__VU * __ITER) % testUsers.length];
  // ...
}
```

---

## 8. Realistic Pacing: Dynamic Think Time & Jitter

Running tests with `sleep(0)` or hardcoded `sleep(1)` creates an artificial synchronized "laser pulse" request wave (micro-bursting) that does not reflect human user behavior.

### Implementing Dynamic Jitter:
```javascript
import { sleep } from 'k6';

// Sleep with uniform random jitter between min and max seconds
export function sleepWithJitter(minSeconds = 1, maxSeconds = 3) {
  const delay = Math.random() * (maxSeconds - minSeconds) + minSeconds;
  sleep(delay);
}
```

---

## In-Depth References
- [`references/script_architecture_and_modular_design.md`](references/script_architecture_and_modular_design.md) - Deep dive into modular architecture, configuration patterns, and scenario journeys.
- [`references/metrics_tags_and_checks.md`](references/metrics_tags_and_checks.md) - Deep dive into custom metrics, tag dimensions, and robust assertions.
- [`references/data_parameterization_and_shared_array.md`](references/data_parameterization_and_shared_array.md) - Deep dive into data generation, CSV/JSON parameterization, and zero-leak memory management.
