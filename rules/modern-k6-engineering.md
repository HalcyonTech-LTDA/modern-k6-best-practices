---
description: Automatically enforces modern Grafana K6 engineering standards: modular script architecture, high-cardinality metric prevention, scenarios and workload modeling, memory hygiene with SharedArray, deterministic quality gates (thresholds), non-halting defensive checks, realistic pacing, zero-secret environment decoupling, official container orchestration, and mandatory living README.md runbook documentation.
trigger: model_decision
---

# Modern Grafana K6 Engineering Guardrails & Skill Router

When authoring, reviewing, refactoring, or executing Grafana K6 performance test suites, workload profiles, thresholds, metric configurations, test data generators, CI/CD pipelines, or performance runbooks, strictly adhere to Modern Grafana K6 Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Script Architecture & Coding** | `modern-k6-test-craftsmanship` | ES6+ modular structure, HTTP client abstraction, low-cardinality metric tagging, custom metrics, and `SharedArray` memory hygiene. |
| **Workload Modeling & Quality Gates** | `modern-k6-workloads-and-thresholds` | Modern `scenarios` API, closed vs. open arrival-rate executors, test profiles (smoke, load, stress), and deterministic thresholds. |
| **Living Documentation & Runbooks** | `modern-k6-living-documentation` | Mandatory dedicated `README.md`, SUT scope, scenario & SLA catalog matrix, environment contracts, and CLI/Docker recipes. |
| **Orchestration, Dashboards & CI/CD** | `modern-k6-orchestration-and-reporting` | Official `grafana/k6` container execution, Taskfile runners, modern Web Dashboard exports, and zero-secret credential injection. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Modular Architecture, Client Wrappers & Low-Cardinality Tagging**: Encapsulate HTTP operations in standardized client helpers with timeouts and correlation headers (`X-Request-ID`). Never issue requests against dynamic, unparameterized URLs without explicit tag grouping (prevent metric cardinality explosion).
2. **Explicit Modern Scenarios Architecture**: Multi-stage and non-trivial performance tests must define explicit `options.scenarios` with tailored executors (model open arrival rates to eliminate Coordinated Omission). Avoid flat global `vus`/`duration` for production test suites.
3. **Strict Memory Hygiene with `SharedArray`**: Never load datasets or parse large JSON fixtures inside per-VU execution iterations. Datasets exceeding 100 records must be initialized once via `SharedArray`.
4. **Deterministic Quality Gates (Thresholds)**: Production and CI suites must declare explicit SLA `thresholds` (latency percentiles and error rates). Breaches must deterministically fail the automated pipeline with a non-zero exit code.
5. **Defensive Parsing & Realistic Pacing**: Never invoke `JSON.parse` without verifying HTTP status codes. Validate assertions using non-halting `check()` calls and inject jittered think times between user requests (ban zero-sleep spam).
6. **Zero Secrets, Containerization & Living Runbooks**: Decouple environment configuration and inject secrets at runtime without hardcoding. Containerize execution via official pinned K6 images. Maintain and synchronize a dedicated living `README.md` in every atomic test commit.

Consult the corresponding skills for complete technical deep dives, configuration recipes, and reference implementations.
