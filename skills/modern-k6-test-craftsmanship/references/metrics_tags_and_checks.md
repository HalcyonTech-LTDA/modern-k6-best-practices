# Metrics, Tags, and Checks in Grafana K6

Metrics, tags, and checks form the measurement foundation of Grafana K6. Understanding their interactions allows engineering teams to construct precise SLA verifications, debug latency bottlenecks, and avoid catastrophic resource exhaustion.

---

## 1. Built-In Metrics Taxonomy

Every HTTP request executed by k6 automatically collects a standard set of built-in metrics:

| Metric | Type | Description |
| :--- | :---: | :--- |
| `http_req_duration` | Trend | Total time for the request (`http_req_sending` + `http_req_waiting` + `http_req_receiving`). |
| `http_req_waiting` | Trend | Time spent waiting for the server's initial response byte (Time to First Byte / TTFB). |
| `http_req_connecting` | Trend | Time spent establishing TCP connection to the remote host. |
| `http_req_tls_handshaking` | Trend | Time spent executing TLS handshake negotiation. |
| `http_req_failed` | Rate | Fraction of requests that failed (based on HTTP status or network errors). |
| `http_reqs` | Counter | Total number of HTTP requests issued by the test. |
| `vus` | Gauge | Current number of active virtual users. |
| `vus_max` | Gauge | Max allocated virtual users for the current scenario. |
| `iterations` | Counter | Total count of completed VU execution loop iterations. |
| `checks` | Rate | Aggregate pass/fail rate across all `check()` statements. |

---

## 2. Preventing High-Cardinality Metric Explosions

### The Cardinality Danger:
By default, k6 attaches the request URL as a tag on HTTP metrics. If your application has parameterized REST paths:
- `/api/v1/orders/10001`
- `/api/v1/orders/10002`
- `/api/v1/orders/10003`

Every distinct URL creates a new distinct metric time-series in memory and downstream metric storage (Prometheus, InfluxDB, Grafana Mimir). If you run 100,000 requests, k6 generates 100,000 unique metric streams, causing **Out of Memory (OOM)** crashes on the runner.

### Pattern 1: Static Name Tagging (Recommended for Client Wrappers)
Assign a normalized endpoint template name via the `tags.name` property:

```javascript
http.get(`${baseUrl}/api/v1/orders/${orderId}`, {
  tags: { name: 'GET /api/v1/orders/:id' },
});
```

### Pattern 2: `http.url` Template Literal
k6 provides an ES6 tagged template literal that automatically treats interpolated parameters as a single grouped URL template:

```javascript
import http from 'k6/http';

http.get(http.url`${baseUrl}/api/v1/orders/${orderId}`);
// k6 tags this metric automatically as: http.url = "https://api.example.com/api/v1/orders/${...}"
```

---

## 3. Custom Metrics: Types & Use Cases

When built-in metrics do not reflect business domain events or sub-transaction times, define custom metrics in the init context:

```javascript
import { Trend, Rate, Counter, Gauge } from 'k6/metrics';

// 1. Trend: Measures timing distributions (percentiles, averages, min, max)
// Pass `true` as the second argument to format as time (milliseconds) in reports
export const paymentGatewayLatency = new Trend('payment_gateway_duration', true);

// 2. Rate: Tracks boolean pass/fail percentages (0.0 to 1.0)
export const fraudCheckApprovedRate = new Rate('fraud_check_approved_rate');

// 3. Counter: Cumulative count of events across the entire test run
export const rateLimitHitCount = new Counter('rate_limit_hits');

// 4. Gauge: Captures point-in-time values (e.g. queue sizes, inventory levels)
// NOTE: All k6 custom metrics (including Gauge) use .add(val). Calling .set(val) throws TypeError!
export const availableWorkerCount = new Gauge('available_workers');
```

### Recording Custom Metrics in Test Code:
```javascript
export default function () {
  const start = Date.now();
  const res = http.post('https://api.example.com/pay', payload);
  const elapsed = Date.now() - start;

  paymentGatewayLatency.add(elapsed);
  fraudCheckApprovedRate.add(res.status === 200);

  if (res.status === 429) {
    rateLimitHitCount.add(1);
  }

  // Record point-in-time gauge value using .add(val)
  availableWorkerCount.add(10);
}
```

---

## 4. Checks: Non-Halting Validations

Unlike traditional unit test assertions that throw an exception and halt execution on failure, `check()` statements in k6 are non-halting:

```javascript
import { check } from 'k6';

const res = client.get('/api/v1/profile');

check(res, {
  'status is 200': (r) => r.status === 200,
  'content type is JSON': (r) => r.headers['Content-Type']?.includes('application/json'),
  'has user profile data': (r) => {
    try {
      const body = JSON.parse(r.body);
      return body && typeof body.email === 'string';
    } catch {
      return false; // Safely handle malformed responses
    }
  },
});
```

### Checks vs Thresholds:
- `check()` only records boolean success or failure. It does **not** fail the test run or trigger a non-zero exit code by itself!
- To make a failing check fail the CI build, you **must bind a threshold** to the `checks` metric:

```javascript
export const options = {
  thresholds: {
    // Fails the test if less than 99% of all checks pass
    checks: ['rate>0.99'],
  },
};
```

---

## 5. Scoped Thresholds Using Metric Tags

Thresholds can be scoped to specific subsets of requests using tag filters:

```javascript
export const options = {
  thresholds: {
    // Global threshold for all requests
    'http_req_duration': ['p(95)<500'],

    // Specific threshold for user login endpoint
    'http_req_duration{name:POST /auth/login}': ['p(95)<250', 'p(99)<400'],

    // Specific threshold for heavy checkout endpoint
    'http_req_duration{name:POST /orders}': ['p(95)<1000'],

    // Specific failure rate threshold for critical flows
    'http_req_failed{flow:checkout}': ['rate<0.005'],
  },
};
```
