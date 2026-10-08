---
name: modern-k6-orchestration-and-reporting
description: Use this skill when orchestrating Grafana K6 executions in containers, CI/CD pipelines, and reporting backends: official grafana/k6 Docker containers, Web Dashboard configuration (K6_WEB_DASHBOARD), custom summary handlers (handleSummary for JUnit/JSON/HTML), telemetry streaming (Grafana Cloud, Prometheus, InfluxDB), and automated GitHub Actions workflows.
---

# Modern Grafana K6 Orchestration & Reporting

This skill governs container orchestration, automated CI/CD pipeline integration, modern reporting dashboards, and telemetry streaming for Grafana K6.

---

## 1. Official Container Orchestration Standards

Running k6 within Docker guarantees reproducible execution across developer workstations and CI agents without requiring local toolchain installation.

### Golden Invariants for Docker Execution:
1. **Never use `:latest`**: Always pin to specific semantic versions (e.g. `grafana/k6:0.54.0`).
2. **Mount test directory as a volume**: Mount tests and reports directory so output artifacts are persisted on the host filesystem.
3. **Respect permissions**: The official `grafana/k6` image runs as an unprivileged user (`k6`, UID 12345). Ensure the host `reports/` directory is writable or created prior to execution.

```bash
# Standard Docker execution command
docker run --rm -i \
  --network="host" \
  -v "$(pwd)/k6:/scripts" \
  -e K6_WEB_DASHBOARD=true \
  -e K6_WEB_DASHBOARD_EXPORT=/scripts/reports/02-load.html \
  -e K6_ENV=local \
  grafana/k6:0.54.0 run /scripts/tests/02-load.js
```

---

## 2. Modern Native Web Dashboard

k6 includes a built-in interactive Web Dashboard capable of displaying real-time graphs and exporting standalone, zero-dependency HTML reports.

### Activation Environment Variables:

| Variable | Type | Description |
| :--- | :---: | :--- |
| `K6_WEB_DASHBOARD` | Boolean | Enables the in-memory web dashboard server (`true` / `false`). |
| `K6_WEB_DASHBOARD_PORT` | Integer | Custom port for live dashboard (defaults to `5665`). |
| `K6_WEB_DASHBOARD_EXPORT` | String | Path to export a self-contained, interactive HTML report file. |
| `K6_WEB_DASHBOARD_OPEN` | Boolean | Automatically opens default browser on test launch. |

### Execution Example:
```bash
K6_WEB_DASHBOARD=true \
K6_WEB_DASHBOARD_PORT=5665 \
K6_WEB_DASHBOARD_EXPORT=k6/reports/02-load-test.html \
k6 run k6/tests/02-load.js
```

The exported HTML file contains all charts, latency percentiles, and threshold outcomes in an interactive bundle that can be archived in CI artifacts or shared with stakeholders.

---

## 3. Custom Summary Handlers (`handleSummary`)

At the conclusion of a test run, k6 invokes the optional `handleSummary(data)` hook in the test script. This hook allows exporting structured machine-readable summaries:

```javascript
// Exporting multiple reports simultaneously: stdout + JSON + JUnit XML
import { jUnit, textSummary } from 'https://jslib.k6.io/k6-summary/0.0.2/index.js';

export function handleSummary(data) {
  return {
    'stdout': textSummary(data, { indent: ' ', enableColors: true }),
    'k6/reports/summary.json': JSON.stringify(data, null, 2),
    'k6/reports/junit.xml': jUnit(data),
  };
}
```

- **JUnit XML**: Ingested natively by GitHub Actions, GitLab CI, and Azure DevOps to render pass/fail test tables in PR comments.
- **JSON**: Enables historical tracking of latency trends across releases by pushing metrics to a data warehouse.

---

## 4. Continuous Integration Pipeline Pattern (GitHub Actions)

Integrate performance tests as an automated gate in CI workflows:

```yaml
# .github/workflows/performance.yml
name: Performance Quality Gate

on:
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  smoke-test:
    name: K6 Smoke Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Application Services
        run: docker compose up -d --wait

      - name: Run K6 Smoke Test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: k6/tests/01-smoke.js
          flags: --env K6_ENV=local
        env:
          K6_WEB_DASHBOARD: 'true'
          K6_WEB_DASHBOARD_EXPORT: 'k6/reports/smoke-report.html'

      - name: Archive Test Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-smoke-report
          path: k6/reports/smoke-report.html
```

---

## In-Depth References
- [`references/container_orchestration_and_docker_runners.md`](references/container_orchestration_and_docker_runners.md) - Deep dive into Docker Compose integration, Taskfile automation, and container security.
- [`references/reporting_dashboards_and_ci_cd.md`](references/reporting_dashboards_and_ci_cd.md) - Deep dive into JUnit XML, Prometheus remote write, Grafana Cloud, and CI/CD quality gate enforcement.
