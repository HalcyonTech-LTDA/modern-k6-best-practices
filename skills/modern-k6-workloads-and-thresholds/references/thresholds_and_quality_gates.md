# Thresholds & Automated Quality Gates

In modern continuous delivery pipelines, performance testing must function as an automated quality gate rather than an observational exercise. Thresholds define programmatic criteria that automatically pass or fail the test execution.

---

## 1. Threshold Syntax & Aggregation Metrics

Threshold expressions consist of an aggregation function, a comparison operator (`<`, `<=`, `>`, `>=`, `==`, `!=`), and a target threshold value:

```javascript
export const options = {
  thresholds: {
    // Trend Metric Aggregations
    'http_req_duration': [
      'avg<150',     // Mean latency must be below 150ms
      'med<100',     // Median (50th percentile) below 100ms
      'p(90)<200',   // 90th percentile latency below 200ms
      'p(95)<300',   // 95th percentile latency below 300ms
      'p(99)<600',   // 99th percentile latency below 600ms
      'max<2000',    // Absolute maximum latency cannot exceed 2000ms
    ],

    // Rate Metric Aggregations (Fraction from 0.0 to 1.0)
    'http_req_failed': [
      'rate<0.01',   // Failed request rate must remain under 1%
    ],

    // Counter Metric Aggregations
    'http_reqs': [
      'count>5000',  // Test must generate at least 5000 total requests
      'rate>50',     // Test must sustain at least 50 requests/sec overall
    ],
  },
};
```

> **Important Unit Note**: In rate metrics (`http_req_failed`, custom `Rate`, `checks`), `0.01` represents 1%, and `0.05` represents 5%. Writing `rate<1` means 100%, which will virtually never trigger!

---

## 2. CI/CD Integration & Non-Zero Exit Codes

When any declared threshold evaluates to `false` at the conclusion of a test run:
- k6 prints a clear failure mark `✗` next to the breached threshold in the CLI output.
- k6 exits with process **Exit Code `99`**.
- Continuous integration runners (GitHub Actions, GitLab CI, Jenkins) detect this non-zero exit code and mark the pipeline step as failed, blocking deployment.

### Handling Threshold Failures in Shell Scripts:
```bash
k6 run tests/02-load.js
EXIT_CODE=$?

if [ $EXIT_CODE -eq 99 ]; then
  echo "Performance Quality Gate FAILED: Threshold breached!"
  exit 1
elif [ $EXIT_CODE -ne 0 ]; then
  echo "Execution Error: Script crashed or network unreachable!"
  exit 1
fi
echo "Performance Quality Gate PASSED!"
```

---

## 3. Tag-Scoped Precision Thresholds

Different endpoints have vastly different expected latencies. A simple health check should complete in under 10ms, while a complex report generation may take 800ms. Applying a blanket `http_req_duration: ['p(95)<200']` leads to false alarms.

Use **tag-scoped thresholds** to enforce targeted SLAs:

```javascript
export const options = {
  thresholds: {
    // Fast lightweight read endpoints
    'http_req_duration{name:GET /actuator/health}': ['p(99)<25'],
    'http_req_duration{name:GET /api/v1/customers/:id}': ['p(95)<150'],

    // Complex transactional write endpoints
    'http_req_duration{name:POST /api/v1/orders}': ['p(95)<400', 'p(99)<800'],

    // Scenario-scoped threshold
    'http_req_duration{scenario:checkout_traffic}': ['p(95)<500'],

    // Status code specific threshold
    'http_req_duration{status:200}': ['p(95)<200'],
  },
};
```

---

## 4. Immediate Circuit Breaking with `abortOnFail`

During stress or spike tests, if a critical backend service or database crashes, letting the test continue for another 15 minutes is wasteful and could cause cascade failures across shared environments.

Use `abortOnFail: true` to terminate the test run immediately upon failure:

```javascript
export const options = {
  thresholds: {
    // If failure rate exceeds 15% after at least 10s of testing, abort immediately!
    http_req_failed: [
      {
        threshold: 'rate<0.15',
        abortOnFail: true,
        delayAbort: '10s', // Gives warm-up period to avoid premature abort
      },
    ],
  },
};
```
