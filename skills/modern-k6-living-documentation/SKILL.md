---
name: modern-k6-living-documentation
description: Use this skill when authoring, updating, or reviewing documentation for Grafana K6 performance testing suites: establishing the mandatory dedicated README.md living documentation, documenting System Under Test (SUT) scope, maintaining scenario & SLA matrices, defining dynamic environment contracts, and enforcing documentation synchronization in pull requests.
---

# Modern Grafana K6 Living Documentation Discipline

This skill defines the non-negotiable standard for maintaining living documentation for Grafana K6 performance test suites. In modern cloud engineering, performance test suites without clear, living documentation decay into unmaintainable, dangerous artifacts that teams fear to run.

---

## 1. The Living Documentation Philosophy

### Why Undocumented Performance Tests Cause System Degradation:
1. **The "Black-Box" Runbook Antipattern**: Performance testing directories frequently accumulate scripts (`load.js`, `test2.js`, `spike_final.js`) with zero documentation on what environment variables they require (`TARGET_URL`, `K6_ENV`, `AUTH_TOKEN`), what traffic intensity they generate, or which microservices they target.
2. **The SLA Amnesia Antipattern**: When thresholds are breached, engineers have no documented context for *why* a specific SLA was established (e.g., P95 < 200ms) or what the acceptable degradation boundary is under peak events.
3. **The Accidental Production Hazard**: Untrained team members running a poorly documented stress script against a staging environment can accidentally trigger unbounded writes, spam payment gateways, or contaminate live databases due to undocumented data strategies.
4. **Silent Contract Drift**: As APIs evolve, unmaintained test scripts silently break or continue testing legacy, deprecated paths while leaving new high-traffic endpoints unverified.

### The Invariant Rule: Dedicated `README.md` Mandate
Whenever a repository contains a Grafana K6 testing suite (e.g. `k6/`, `performance/`, `tests/load/`), it **MUST** maintain a dedicated, living `README.md` located directly in that root testing directory.

Any pull request or commit that introduces, alters, or removes a test script, endpoint journey, workload stage, environment variable, or SLA threshold **MUST** synchronize this `README.md` within the exact same atomic commit.

---

## 2. Anatomy of a Production-Grade K6 `README.md`

A professional K6 `README.md` consists of eight mandatory sections:

```text
k6/README.md
├── 1. Overview & Service Under Test (SUT) Scope
├── 2. Directory Architecture Tree
├── 3. Target Endpoints & Business Scenarios Catalog
├── 4. Environment Selection & Configuration Matrix
├── 5. Workload Profiles & Quality Gates (SLO Matrix)
├── 6. Execution Runbook (Taskfile / Make / CLI / Docker)
├── 7. Observability, Web Dashboards & HTML Reports
└── 8. Test Data Strategy & Safety Runbook
```

### Detailed Breakdown of Sections:

### 1. Overview & Service Under Test (SUT) Scope
- Explicitly states which service, microservice, or cluster is under test.
- Links to relevant application entrypoints (controllers, API gateway routes).
- Clarifies testing goals (capacity planning, latency benchmarking, regression prevention).

### 2. Directory Architecture Tree
- Provides a clean ASCII tree detailing every directory and file role:
  - `config/` (environment mappings)
  - `helpers/` (client wrappers, data generators, metrics)
  - `scenarios/` (reusable business flows)
  - `tests/` (runnable suites: smoke, load, stress, spike, soak)
  - `reports/` (dashboard exports)

### 3. Target Endpoints & Business Scenarios Catalog
- Catalogs all simulated user journeys:
  - HTTP Verb & Path
  - Business transaction description
  - Expected HTTP status codes (success and controlled error paths)
  - Correlation IDs and headers used

### 4. Environment Selection & Configuration Matrix
- Documents all supported environments and their default base URLs:
  - `local` (default): `http://localhost:8080` (or `http://host.docker.internal:8080` for Docker)
  - `dev`: `https://dev-api.example.com`
  - `staging`: `https://staging-api.example.com`
  - `prod`: `https://api.example.com` (canary verification)
- Documents environment variables (`K6_ENV`, `TARGET_URL`, `AUTH_TOKEN`, `K6_WEB_DASHBOARD`).

### 5. Workload Profiles & Quality Gates (SLO Matrix)
A comprehensive markdown table cataloging all runnable test suites:
- **Test Profile Name**: (e.g., Smoke, Load, Stress, Spike, Soak)
- **Workload Shape**: VU count, ramp stages, duration, executor type
- **Target SLOs / Thresholds**: P95, P99, error rate limits
- **Runnable Command**: CLI or Taskfile invocation

### 6. Execution Runbook
- Clear, copy-pasteable execution recipes covering:
  - **Local CLI**: `k6 run k6/tests/01-smoke.js`
  - **Task runner**: `task k6:load K6_ENV=staging`
  - **Containerized execution (Docker fallback)**: Running via `grafana/k6` container when the CLI is not locally installed.

### 7. Observability, Web Dashboards & HTML Reports
- Instructions for enabling the native K6 Web Dashboard:
  - `K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=k6/reports/02-load.html`
- How to interpret exported metrics (percentiles, TTFB, RPS, threshold compliance).
- Integration with external metrics collectors (Prometheus, Grafana Cloud, InfluxDB).

### 8. Test Data Strategy & Safety Runbook
- Details whether tests use dynamic synthetic data or pre-seeded fixtures.
- Safety guidelines preventing destructive operations or unintended side effects in shared environments.

---

## 3. Pull Request Synchronization Checklist

Before submitting or approving any pull request that touches performance testing code, verify:

- [ ] Does `README.md` reflect all newly added or modified endpoints?
- [ ] Are all new environment variables documented with sample values?
- [ ] Is the SLO / Thresholds matrix updated if latency limits or error rates were adjusted?
- [ ] Are all execution commands in the Runbook copy-pasteable and tested?
- [ ] Are report export paths and dashboard instructions accurate?

---

## In-Depth References
- [`references/k6_readme_standard_template.md`](references/k6_readme_standard_template.md) - Complete, domain-agnostic template blueprint for a production-grade K6 `README.md`.
- [`references/sla_matrix_and_runbook_recipes.md`](references/sla_matrix_and_runbook_recipes.md) - Deep dive into constructing scenario matrices, SLA quality gate tables, and Docker fallback execution commands.
