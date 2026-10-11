# Modern Grafana K6 Best Practices Plugin

An Antigravity plugin providing automated engineering guardrails, architectural patterns, workload modeling standards, container orchestration, and procedural skills for modern **Grafana K6** performance testing: modular ES6+ script craftsmanship, high-cardinality metric prevention, scenarios and arrival-rate executors (eliminating Coordinated Omission), memory-efficient `SharedArray` hygiene, deterministic quality gates (thresholds), native Web Dashboards, and the **mandatory living `README.md` documentation discipline**.

---

## Installation

### Antigravity

Install directly in Antigravity via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-k6-best-practices
```

Verify that the plugin is installed and ready:

```bash
agy plugin list
```

### Claude Code

Add the Halcyon Tech marketplace once, then install the plugin:

```text
/plugin marketplace add HalcyonTech-LTDA/halcyon-plugins
/plugin install modern-k6-best-practices@halcyon-plugins
```

Skills are namespaced by plugin in Claude Code (`/modern-k6-best-practices:<skill-name>`). The guardrails in `rules/` load through the skills in `claude/skills/`.

---
## Overview

In modern cloud-native systems, continuous delivery, and distributed microservices, performance testing is not an occasional manual benchmark. It is a continuous, automated quality gate.

However, performance testing implementations frequently suffer from critical engineering antipatterns:
- **Monolithic, Unmaintainable Scripts**: 1,000-line single-file scripts combining options, configuration, payloads, HTTP calls, and assertions.
- **High-Cardinality Metric Explosions**: Unchecked dynamic URLs (e.g. `/orders/${id}`) generating millions of metrics dimensions that crash load generators and downstream Prometheus/Mimir backends.
- **Coordinated Omission**: Relying strictly on VU-based closed models (`ramping-vus`) where system latency slows down the request rate, creating a false illusion of stability under stress.
- **Memory Starvation**: Duplicating megabytes of CSV/JSON data fixtures across hundreds of virtual users instead of leveraging read-only `SharedArray` memory.
- **The "Black-Box" Test Antipattern**: Undocumented test folders where teams do not know what endpoints are being tested, which environment variables configure them, or how to execute suites locally.

This plugin establishes modern, production-grade engineering standards for Grafana K6 across four foundational pillars:

1. **Modular Script Craftsmanship & Defensive Coding**: Structuring tests into decoupled layers (config, client helpers, synthetic data generators, user journey flows, and runnable test suites), eliminating metric cardinality explosions with tag grouping, handling JSON parsing defensively, and utilizing non-halting checks.
2. **Workload Modeling & Deterministic Quality Gates**: Leveraging the modern `scenarios` API, choosing between closed (VU-based) and open (arrival-rate) models to eliminate Coordinated Omission, maintaining five standardized test profiles (Smoke, Load, Stress, Spike, Soak), and establishing automated CI/CD threshold quality gates.
3. **Mandatory Living Documentation**: Requiring every performance testing directory to maintain an up-to-date, comprehensive `README.md` acting as a living runbook with System Under Test (SUT) scope, scenario & SLA matrices, dynamic environment contracts, execution recipes, and PR synchronization checklists.
4. **Container Orchestration & Modern Observability**: Standardizing official `grafana/k6` container execution with intelligent Docker fallbacks (Taskfile/Makefile), configuring native real-time Web Dashboards (`K6_WEB_DASHBOARD`), and exporting structured summaries (JUnit XML, JSON).

---

## What's Included

### 1. Automatic Guardrails (`rules/modern-k6-engineering.md`)
Automatically enforced across all projects utilizing Grafana K6:
- **Mandatory Dedicated `README.md`**: Absolute mandate requiring any directory housing k6 scripts to maintain an up-to-date living runbook synchronized in every PR.
- **Ban on High-Cardinality Metric Tagging**: Mandatory URL template grouping (`tags: { name: '...' }` or `http.url`) to prevent metric explosion and OOM crashes.
- **Mandatory Modern `scenarios` Architecture**: Prohibiting flat global options for multi-workload tests in favor of the `scenarios` API.
- **Strict Memory Hygiene with `SharedArray`**: Requiring `k6/data` `SharedArray` for external fixtures > 100 rows to prevent per-VU RAM duplication.
- **Deterministic Quality Gates (Thresholds)**: Mandatory pass/fail thresholds on latency percentiles (P90/P95/P99) and error rates (`http_req_failed`) exiting with code 99 on breach.
- **Defensive Parsing & Non-Halting Checks**: Requiring safe status validation before `JSON.parse` and using `check()` rather than throwing fatal exceptions.
- **Realistic Pacing & Dynamic Think Time**: Banning zero-delay hammering in user journeys; requiring randomized think time with jitter to prevent laser-burst microbursting.
- **Zero Secrets & Dynamic Environment Decoupling**: Centralizing environment resolution via `config/environments.js` and banning hardcoded credentials or URLs.
- **Standardized Resilient HTTP Client Wrapper**: Encapsulating HTTP requests with default timeouts, headers, and distributed tracing correlation IDs (`X-Correlation-ID`).
- **Containerized Execution & Self-Contained Automation**: Providing containerized execution via `grafana/k6` and automated runners with intelligent Docker fallback.

---

### 2. Specialized Skills (`skills/`)

#### 🛠️ `modern-k6-test-craftsmanship`
Guides modular script architecture, client wrappers, metric tagging, and data parameterization:
- `references/script_architecture_and_modular_design.md`: Five-layer directory architecture, configuration resolution, decoupled user journey flow pattern, and composition of test suites.
- `references/metrics_tags_and_checks.md`: Built-in metric taxonomy, URL tag grouping strategies, custom metrics (Trend, Rate, Counter, Gauge), non-halting checks, and tag-scoped thresholds.
- `references/data_parameterization_and_shared_array.md`: Dynamic synthetic data generators vs. seeded fixtures, `SharedArray` memory architecture, and JSON/CSV streaming patterns.

#### ⚖️ `modern-k6-workloads-and-thresholds`
Guides workload modeling, executor selection, test profiles, and SLA quality gates:
- `references/scenarios_and_executors.md`: The 7 K6 executors matrix, open vs closed models, eliminating Coordinated Omission with arrival rates, and multi-scenario scheduling.
- `references/test_profiles_and_workload_patterns.md`: The five standard test profiles (Smoke, Load, Stress, Spike, Soak) with runnable options configurations and operational goals.
- `references/thresholds_and_quality_gates.md`: Threshold syntax, percentile math, tag filtering, non-zero exit codes (exit 99) in CI/CD, and `abortOnFail` circuit breakers.

#### 📖 `modern-k6-living-documentation`
Guides the mandatory living documentation standard for Grafana K6 testing suites:
- `references/k6_readme_standard_template.md`: Complete, production-grade template blueprint for a dedicated `k6/README.md`.
- `references/sla_matrix_and_runbook_recipes.md`: Designing SLA/SLO quality gate matrices, Taskfile/Makefile runners with intelligent Docker fallback, cross-platform container networking, and PR synchronization checklists.

#### 🐳 `modern-k6-orchestration-and-reporting`
Guides containerization, Web Dashboards, CI/CD quality gates, and telemetry streaming:
- `references/container_orchestration_and_docker_runners.md`: Official `grafana/k6` Docker images, non-root user permissions (UID 12345), Docker Compose choreography, and custom `xk6` extensions.
- `references/reporting_dashboards_and_ci_cd.md`: Native Web Dashboard configuration (`K6_WEB_DASHBOARD`), standalone HTML exports, custom `handleSummary` reporting (JUnit XML, JSON), Prometheus remote write, and GitHub Actions pipelines.

---

## Directory Layout

```text
modern-k6-best-practices/
├── plugin.json
├── README.md
├── rules/
│   └── modern-k6-engineering.md
└── skills/
    ├── modern-k6-test-craftsmanship/
    │   ├── SKILL.md
    │   └── references/
    │       ├── data_parameterization_and_shared_array.md
    │       ├── metrics_tags_and_checks.md
    │       └── script_architecture_and_modular_design.md
    ├── modern-k6-workloads-and-thresholds/
    │   ├── SKILL.md
    │   └── references/
    │       ├── scenarios_and_executors.md
    │       ├── test_profiles_and_workload_patterns.md
    │       └── thresholds_and_quality_gates.md
    ├── modern-k6-living-documentation/
    │   ├── SKILL.md
    │   └── references/
    │       ├── k6_readme_standard_template.md
    │       └── sla_matrix_and_runbook_recipes.md
    └── modern-k6-orchestration-and-reporting/
        ├── SKILL.md
        └── references/
            ├── container_orchestration_and_docker_runners.md
            └── reporting_dashboards_and_ci_cd.md
```

---

## License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details.
Copyright (c) 2026 Halcyon Tech.
