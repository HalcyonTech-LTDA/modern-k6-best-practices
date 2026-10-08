# Reporting Dashboards, Telemetry & CI/CD Quality Gates

Modern performance engineering treats metrics as observable telemetry rather than ephemeral terminal output. Grafana K6 provides native interactive dashboards, structured summary exporters, and time-series telemetry backends.

---

## 1. Native Web Dashboard Architecture

The Grafana K6 Web Dashboard is a zero-dependency, browser-based UI built directly into the k6 Go binary:

```text
 ┌─────────────────────────────────────────────────────────────┐
 │                         k6 Runtime                          │
 │  ┌─────────────────┐                 ┌───────────────────┐  │
 │  │ Virtual Users   │ ──Metrics───>   │ In-Memory Buffer  │  │
 │  └─────────────────┘                 └─────────┬─────────┘  │
 └────────────────────────────────────────────────┼────────────┘
                                                  │
                 ┌────────────────────────────────┴────────────────────────────────┐
                 │                                                                 │
                 ▼ (Live Mode: Port 5665)                                          ▼ (At Test End)
     ┌───────────────────────┐                                         ┌───────────────────────┐
     │  Live Browser View    │                                         │ Self-Contained HTML   │
     │  http://localhost:5665│                                         │ reports/02-load.html  │
     └───────────────────────┘                                         └───────────────────────┘
```

### Configuration Flags & Environment Variables:
```bash
# 1. Live dashboard while test runs:
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_PORT=5665 k6 run tests/02-load.js

# 2. Export standalone HTML report at end:
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=reports/02-load.html k6 run tests/02-load.js

# 3. Headless export (no live server, only output file generated):
k6 run --out web-dashboard=export=reports/02-load.html tests/02-load.js
```

---

## 2. Multi-Format Summary Export via `handleSummary`

The `handleSummary` callback executes in the init context after the test finishes and all metrics are aggregated. It can return a map of filename to serialized content:

```javascript
// helpers/summary-reporter.js
export function generateCustomReports(data) {
  const result = {};

  // 1. Machine-readable JSON for performance regression tracking
  result['k6/reports/summary.json'] = JSON.stringify({
    timestamp: new Date().toISOString(),
    metrics: {
      http_req_duration_p95: data.metrics.http_req_duration?.values['p(95)'],
      http_req_duration_avg: data.metrics.http_req_duration?.values.avg,
      http_req_failed_rate: data.metrics.http_req_failed?.values.rate,
      total_requests: data.metrics.http_reqs?.values.count,
    },
    root_group: data.root_group,
  }, null, 2);

  // 2. Terminal summary output to stdout
  result['stdout'] = formatTextSummary(data);

  return result;
}

function formatTextSummary(data) {
  const p95 = data.metrics.http_req_duration?.values['p(95)']?.toFixed(2) || 'N/A';
  const failRate = ((data.metrics.http_req_failed?.values.rate || 0) * 100).toFixed(2);
  return `
========================================
       PERFORMANCE TEST SUMMARY
========================================
- P95 Response Time: ${p95} ms
- Failure Rate:      ${failRate} %
- Total Requests:    ${data.metrics.http_reqs?.values.count || 0}
========================================
`;
}
```

---

## 3. Streaming Telemetry to Prometheus & Grafana Mimir

For enterprise observability, stream metrics in real time to Prometheus or Grafana Cloud via Prometheus Remote Write:

```bash
K6_PROMETHEUS_REMOTE_URL="http://prometheus:9090/api/v1/write" \
K6_PROMETHEUS_REMOTE_MAPPING_CONFIG="mapping.json" \
k6 run -o experimental-prometheus-rw tests/02-load.js
```

### Streaming to InfluxDB:
```bash
k6 run --out influxdb=http://localhost:8086/k6_metrics tests/02-load.js
```

---

## 4. GitHub Actions Automated Quality Gate

Embed performance checks directly into pull requests. If thresholds fail, the CI run fails and uploads the interactive HTML report as an artifact:

```yaml
# .github/workflows/perf-check.yml
name: Performance Quality Gate

on:
  pull_request:
    paths:
      - 'src/**'
      - 'k6/**'

jobs:
  performance-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Spin up target stack
        run: docker compose up -d --wait

      - name: Run K6 Smoke Suite
        run: |
          mkdir -p k6/reports
          docker run --rm \
            --network="host" \
            -v "$(pwd)/k6:/scripts" \
            -e K6_WEB_DASHBOARD=true \
            -e K6_WEB_DASHBOARD_EXPORT=/scripts/reports/smoke-report.html \
            -e K6_ENV=local \
            grafana/k6:0.54.0 run /scripts/tests/01-smoke.js

      - name: Publish Performance Report Artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-smoke-report
          path: k6/reports/smoke-report.html
```
