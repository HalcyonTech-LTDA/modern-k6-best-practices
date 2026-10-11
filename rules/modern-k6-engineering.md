---
description: Automatically enforces modern Grafana K6 engineering standards: modular script architecture, high-cardinality metric prevention, scenarios and workload modeling, memory hygiene with SharedArray, deterministic quality gates (thresholds), non-halting defensive checks, realistic pacing, zero-secret environment decoupling, official container orchestration, and mandatory living README.md runbook documentation.
trigger: model_decision
---

# Modern Grafana K6 Engineering Guardrails & Skill Router

When authoring, reviewing, refactoring, or executing Grafana K6 performance test suites, workload profiles, thresholds, metric configurations, test data generators, CI/CD pipelines, or performance runbooks, strictly adhere to Modern Grafana K6 Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Script Architecture & Coding** | `modern-k6-test-craftsmanship` | ES6+ modular structure, HTTP client abstraction, URL metric tag grouping (preventing cardinality explosion), custom metrics (Trend, Rate, Counter, Gauge), defensive parsing, and `SharedArray` memory hygiene. |
| **Workload Modeling & Quality Gates** | `modern-k6-workloads-and-thresholds` | Modern `scenarios` API, executor selection (closed vs. open arrival-rate models to prevent Coordinated Omission), test profiles (smoke, load, stress, spike, soak), and deterministic SLA thresholds. |
| **Living Documentation & Runbooks** | `modern-k6-living-documentation` | Mandatory dedicated `README.md`, System Under Test (SUT) scope, scenario & SLA catalog matrix, dynamic environment variable contracts, CLI/Docker recipes, and PR synchronization checklists. |
| **Orchestration, Dashboards & CI/CD** | `modern-k6-orchestration-and-reporting` | Official `grafana/k6` container execution, Taskfile/Makefile runners with Docker fallback, modern Web Dashboard (`K6_WEB_DASHBOARD`), summary exporters (`handleSummary`), and zero-secret credential injection. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Mandatory Dedicated `README.md` (Living Runbook Mandate)**:
   - Every directory or project module housing Grafana K6 performance test suites (e.g., `k6/`, `performance/`, `tests/load/`) MUST maintain a dedicated, up-to-date `README.md` located directly in that directory.
   - Any pull request or commit that introduces, alters, or removes test scripts, endpoints, workload profiles, environment configs, thresholds, or execution options MUST synchronize this `README.md` within the exact same commit.
   - The `README.md` must document:
     - The target System Under Test (SUT) scope and endpoint catalog.
     - Workload profile catalog (Smoke, Load, Stress, Spike, Soak) with VU stages, target durations, and rationale.
     - Environment selection matrix and environment variables (`K6_ENV`, `TARGET_URL`, auth tokens).
     - Execution runbook covering both CLI and containerized execution (Docker fallback, Taskfile/Makefile).
     - Thresholds, SLOs, and quality gates matrix (P90, P95, P99, error rates).
     - Reporting mechanisms (Web Dashboard, exported HTML/JSON artifacts, external metrics backends).
     - Synthetic data generation and production safety guidelines.

2. **Absolute Ban on High-Cardinality Dynamic URL Tagging (Metric Explosion Prevention)**:
   - Never issue HTTP requests using raw parameterized or dynamic URLs without explicit tag grouping (e.g. `http.get('/customers/' + id)` without a grouped tag creates a new metric timeseries for every unique ID).
   - Either pass `tags: { name: 'GET /customers/:id' }` in request params, or use the `http.url` template tag literal: ``http.get(http.url`http://example.com/customers/${id}`)``.
   - High cardinality exhausts load generator memory and overwhelms downstream time-series storage (Prometheus, InfluxDB, Grafana Mimir).

3. **Mandatory Modern `scenarios` Architecture over Deprecated Global Options**:
   - For any multi-stage, multi-flow, or non-trivial performance test, declare explicit `options.scenarios` with named scenarios and tailored executors (`ramping-vus`, `constant-arrival-rate`, `shared-iterations`, etc.).
   - Avoid legacy flat global `vus` and `duration` for production suites (permissible only for minimal single-flow smoke tests).
   - Model decoupled arrival rates (`constant-arrival-rate`, `ramping-arrival-rate`) when testing open systems to eliminate Coordinated Omission.

4. **Strict Memory Hygiene with `SharedArray` for Large Datasets**:
   - Never load large test fixture files or data arrays inside default execution functions or per-VU loops using raw `JSON.parse(open(...))` or global arrays.
   - Datasets > 100 rows MUST be loaded once in the init context using `new SharedArray('name', () => JSON.parse(open('./data.json')))` from `k6/data`.
   - Prevents memory duplication across hundreds of concurrent VUs which can crash the load generator machine.

5. **Deterministic Thresholds (Actionable Quality Gates)**:
   - Every production, load, stress, or CI test MUST declare deterministic `thresholds` in `options` covering at least response latency percentiles (`http_req_duration: ['p(95)<300']`) and failure rate (`http_req_failed: ['rate<0.01']`).
   - Smoke tests must assert zero tolerance for unhandled errors.
   - In automated CI/CD pipelines, threshold breaches must deterministically return a non-zero exit code to fail the build gate.
   - Use tag-scoped thresholds (e.g. `'http_req_duration{name:OrderCheckout}': ['p(99)<800']`) for critical user journeys.

6. **Defensive Response Parsing & Non-Halting Checks**:
   - Never invoke `JSON.parse(res.body)` blindly without first verifying HTTP status code or checking for non-empty bodies. Parsing invalid or HTML error bodies crashes the VU with an uncaught exception.
   - Use `check()` for functional validations rather than throwing exceptions or halting execution, allowing aggregate success rates (`checks`) to be tracked in metrics.
   - Pair checks with thresholds (e.g. `'checks': ['rate>0.99']`) to enforce automated test pass/fail.

7. **Realistic Pacing & Dynamic Think Time (Ban on Artificial Denial of Service)**:
   - Never run user journey simulations with zero pause (`sleep(0)`) between requests unless explicitly testing raw saturated throughput via arrival-rate executors.
   - In user behavior simulations, inject realistic, jittered think time (e.g. `sleep(Math.random() * 2 + 1)`) or Poisson distribution pacing.
   - Use dynamic, unique synthetic data generation (unique emails, IDs, tokens) rather than repeating identical static payloads to avoid artificial cache hits.

8. **Zero Hardcoded Secrets & Dynamic Environment Decoupling**:
   - Never hardcode target URLs, credentials, API keys, or bearer tokens into test source files.
   - Centralize environment configuration into a `config/environments.js` module driven dynamically by `__ENV.K6_ENV`, `__ENV.TARGET_URL`, and secret environment variables (`__ENV.AUTH_TOKEN`).
   - Provide a `.env.example` documenting all configuration keys.

9. **Standardized HTTP Client Wrapper & Correlation Tracing**:
   - Never duplicate raw `http.get` / `http.post` calls with scattered header setups across test files.
   - Encapsulate HTTP requests in a standardized client helper that automatically injects default timeouts, `Content-Type: application/json`, `User-Agent: k6-load-test`, and distributed trace headers (`X-Request-ID` or `X-Correlation-ID: uuidv4()`).

10. **Containerized Execution & Self-Contained Automation**:
    - Performance test suites must provide seamless containerized execution via the official `grafana/k6` image (pinned version, never `:latest`).
    - Provide automated task recipes (e.g. `Taskfile.yaml` or `Makefile`) that automatically fallback to Docker when the local `k6` binary is not found on the host machine.
    - Standardize report exports using modern K6 Web Dashboard (`K6_WEB_DASHBOARD=true`, `K6_WEB_DASHBOARD_EXPORT=reports/<test>.html`) and JSON/JUnit summaries.

Consult the corresponding skills for complete technical deep dives, configuration recipes, and reference implementations.
