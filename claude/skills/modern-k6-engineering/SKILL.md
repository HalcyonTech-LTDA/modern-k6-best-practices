---
name: modern-k6-engineering
description: "Automatically enforces modern Grafana K6 engineering standards: modular script architecture, high-cardinality metric prevention, scenarios and workload modeling, memory hygiene with SharedArray, deterministic quality gates (thresholds), non-halting defensive checks, realistic pacing, zero-secret environment decoupling, official container orchestration, and mandatory living README.md runbook documentation."
---

# Modern Grafana K6 Engineering Guardrails & Skill Router

Entry point for this plugin's guardrails in Claude Code. Before acting, read the full standard:

`${CLAUDE_PLUGIN_ROOT}/rules/modern-k6-engineering.md`

It routes each kind of task to the specialized skill of this plugin and lists the non-negotiable rules.
In Claude Code those skills are namespaced: invoke `modern-k6-best-practices:<skill-name>`.

> Generated from `rules/modern-k6-engineering.md`, which is also the Antigravity rule. Edit the rule, not this file.
