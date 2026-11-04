<div align="center">

# Modern K6 Best Practices Plugin
### Modern Performance Testing, Automated SLA Thresholds & Load Simulation

[![Antigravity Plugin](https://img.shields.io/badge/Antigravity-Plugin%20v1.0.0-4F46E5.svg)](https://antigravity.google)
[![License: MIT](https://img.shields.io/badge/License-MIT-10B981.svg)](./LICENSE)
[![Organization](https://img.shields.io/badge/Organization-HalcyonTech--LTDA-black.svg)](https://github.com/HalcyonTech-LTDA)

<p align="center">
  <b>Professional modern Grafana K6 performance engineering best practices covering modular script craftsmanship, workload modeling & scenarios, deterministic thresholds & quality gates, containerized orchestration & reporting, and mandatory living README.md runbook documentation discipline.</b>
</p>

</div>

---

## Installation & Verification

### Antigravity CLI (`agy`)

Install directly into your Antigravity environment via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-k6-best-practices
```

Verify that the plugin is active and adheres to the strict official schema:

```bash
agy plugin list
agy plugin validate --strict ./modern-k6-best-practices
```

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

---

## What's Included (Component Catalog)

### Contextual Rules (`rules/`)
Automatically loaded to govern and protect project contexts:
- **[`modern-k6-engineering.md`](rules/modern-k6-engineering.md)**: Automatically enforces modern Grafana K6 engineering standards: modular script architecture, high-cardinality metric prevention, scenarios and workload modeling, memory hygiene with SharedArray, deterministic quality gates (thresholds), non-halting defensive checks, realistic pacing, zero-secret environment decoupling, official container orchestration, and mandatory living README.md runbook documentation.

### Procedural Skills (`skills/`)
On-demand actionable workflows and runbooks:
- **[`/modern-k6-living-documentation`](skills/modern-k6-living-documentation/SKILL.md)**: Use this skill when authoring, updating, or reviewing documentation for Grafana K6 performance testing suites: establishing the mandatory dedicated README.md living documentation, documenting System Under Test (SUT) scope, maintaining scenario & SLA matrices, defining dynamic environment contracts, and enforcing documentation synchronization in pull requests.
- **[`/modern-k6-orchestration-and-reporting`](skills/modern-k6-orchestration-and-reporting/SKILL.md)**: Use this skill when orchestrating Grafana K6 executions in containers, CI/CD pipelines, and reporting backends: official grafana/k6 Docker containers, Web Dashboard configuration (K6_WEB_DASHBOARD), custom summary handlers (handleSummary for JUnit/JSON/HTML), telemetry streaming (Grafana Cloud, Prometheus, InfluxDB), and automated GitHub Actions workflows.
- **[`/modern-k6-test-craftsmanship`](skills/modern-k6-test-craftsmanship/SKILL.md)**: Use this skill when authoring, structuring, or refactoring Grafana K6 test scripts: modern ES6+ modular architecture, standardized HTTP client abstraction, URL metric tag grouping to prevent high cardinality, custom metrics (Trend, Rate, Counter, Gauge), defensive response parsing, non-halting checks, and memory-efficient SharedArray parameterization.
- **[`/modern-k6-workloads-and-thresholds`](skills/modern-k6-workloads-and-thresholds/SKILL.md)**: Use this skill when modeling workloads and configuring quality gates in Grafana K6: modern scenarios API, executor selection (closed vs open arrival-rate models to prevent Coordinated Omission), workload profiles (smoke, load, stress, spike, soak), and deterministic SLA thresholds.

---

## Directory Hierarchy

```text
modern-k6-best-practices/
├── plugin.json              # Official Antigravity manifest (schema v1)
├── LICENSE                  # MIT License (Copyright 2026 HalcyonTech LTDA)
├── .gitignore               # System ignores & installed_version.json exclusions
├── .gitattributes           # Deterministic line endings (* text=auto eol=lf)
├── README.md                # Golden Standard documentation & runbook
├── assets/
│   └── logo.svg             # Official Halcyon Tech corporate vector symbol
├── rules/                   # Contextual rules discovered automatically
│   └── modern-k6-engineering.md
├── skills/                  # Actionable procedural skills (Progressive Disclosure)
│   └── modern-k6-living-documentation/
│       └── SKILL.md
│   └── modern-k6-orchestration-and-reporting/
│       └── SKILL.md
│   └── modern-k6-test-craftsmanship/
│       └── SKILL.md
│   └── modern-k6-workloads-and-thresholds/
│       └── SKILL.md
```

---

## 💡 Prompt Recipes & Use Cases

```markdown
- "Design an automated k6 performance testing suite with custom thresholds"
- "Structure stress, spike, and soak load tests for high-throughput APIs"
- "Analyze k6 test execution metrics and identify system latency bottlenecks"
```

---

## 📄 License & Corporate Governance

Distributed under the **MIT License**. Copyright (c) 2026 **HalcyonTech LTDA**.  
Engineered with precision and excellence by [Halcyon Tech](https://github.com/HalcyonTech-LTDA).
