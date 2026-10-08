# SLA Matrices & Runbook Automation Recipes

This guide covers constructing production-grade SLA quality gate matrices, implementing automated Taskfile/Makefile runners with intelligent Docker fallback, and managing platform-specific container networking.

---

## 1. Constructing the SLA & Quality Gate Matrix

A performance test is meaningless without unambiguous Service Level Objectives (SLOs). Documenting these criteria in a structured matrix aligns engineering, product, and SRE teams on acceptable performance boundaries.

### Production SLA Matrix Example:

| Transaction / Endpoint | Target Volume | P90 Max | P95 Max | P99 Max | Max Error Rate | Threshold Definition |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Health Check** (`GET /health`) | 100 RPS | 10ms | 20ms | 50ms | 0.0% | `'http_req_duration{name:GET /health}': ['p(99)<50']` |
| **Catalog Search** (`GET /search`) | 150 RPS | 80ms | 150ms | 300ms | < 0.5% | `'http_req_duration{name:GET /search}': ['p(95)<150']` |
| **Customer Lookup** (`GET /customers/:id`) | 80 RPS | 50ms | 100ms | 250ms | < 0.1% | `'http_req_duration{name:GET /customers/:id}': ['p(95)<100']` |
| **Order Placement** (`POST /orders`) | 30 RPS | 200ms | 350ms | 800ms | < 1.0% | `'http_req_duration{name:POST /orders}': ['p(95)<350']` |
| **Aggregate Health** (All requests) | Combined | — | 300ms | — | < 1.0% | `'http_req_failed': ['rate<0.01']` |

---

## 2. Automated Runbook via Taskfile with Intelligent Docker Fallback

A common developer friction point is requiring every contributor to manually install the `k6` CLI via Homebrew, apt, or choco.

By implementing an automated **Taskfile** (or Makefile) with an intelligent fallback, developers who have `k6` installed enjoy instant execution, while developers who do not have `k6` run the exact same tests via Docker seamlessly without manual setup.

### Taskfile.yaml Pattern:

```yaml
version: '3'

vars:
  K6_IMAGE: 'grafana/k6:0.54.0'
  K6_DIR: 'k6'

tasks:
  k6:smoke:
    desc: 'Runs baseline k6 smoke test (auto-detects local CLI vs Docker)'
    vars:
      ENV: '{{.K6_ENV | default "local"}}'
      REPORT: '{{.K6_DIR}}/reports/01-smoke.html'
    cmds:
      - |
        if command -v k6 >/dev/null 2>&1; then
          echo "==> Running k6 smoke test via local CLI..."
          K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT={{.REPORT}} \
            k6 run -e K6_ENV={{.ENV}} {{.K6_DIR}}/tests/01-smoke.js
        else
          echo "==> k6 CLI not found. Falling back to official Docker container..."
          docker run --rm -i \
            --network="host" \
            -v "$(pwd)/{{.K6_DIR}}:/scripts" \
            -e K6_WEB_DASHBOARD=true \
            -e K6_WEB_DASHBOARD_EXPORT=/scripts/reports/01-smoke.html \
            -e K6_ENV={{.ENV}} \
            {{.K6_IMAGE}} run /scripts/tests/01-smoke.js
        fi

  k6:load:
    desc: 'Runs sustained k6 load test'
    vars:
      ENV: '{{.K6_ENV | default "local"}}'
      REPORT: '{{.K6_DIR}}/reports/02-load.html'
    cmds:
      - |
        if command -v k6 >/dev/null 2>&1; then
          K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT={{.REPORT}} \
            k6 run -e K6_ENV={{.ENV}} {{.K6_DIR}}/tests/02-load.js
        else
          docker run --rm -i \
            --network="host" \
            -v "$(pwd)/{{.K6_DIR}}:/scripts" \
            -e K6_WEB_DASHBOARD=true \
            -e K6_WEB_DASHBOARD_EXPORT=/scripts/reports/02-load.html \
            -e K6_ENV={{.ENV}} \
            {{.K6_IMAGE}} run /scripts/tests/02-load.js
        fi
```

---

## 3. Cross-Platform Docker Networking Caveats

When falling back to Docker for local testing, container networking varies by host operating system:

### Linux Hosts:
`--network="host"` enables the container to directly access `http://localhost:8080` running on the host machine.

### macOS & Windows (Docker Desktop):
`--network="host"` behaves differently on macOS/Windows due to the lightweight VM layer.
- `localhost` inside the container resolves to the container itself, not the host machine!
- Use `http://host.docker.internal:8080` to target services running on the host machine.
- Your `config/environments.js` should dynamically handle this:

```javascript
// config/environments.js
const isDocker = __ENV.RUNNING_IN_DOCKER === 'true';

export const ENVIRONMENTS = {
  local: {
    baseUrl: isDocker ? 'http://host.docker.internal:8080' : 'http://localhost:8080',
    timeout: '5s',
  },
};
```
