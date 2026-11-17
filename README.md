# Global O11y Templates

Scenario templates for the Elastic Observability Chaos Demo platform. Each scenario generates realistic synthetic telemetry (metrics, logs, traces) for a specific cloud provider's network operations stack, enabling end-to-end demos of Elastic's observability, AI assistant, and incident response capabilities.

## Table of Contents

- [Overview](#overview)
- [Scenarios](#scenarios)
- [Directory Structure](#directory-structure)
- [Schema DSL Reference](#schema-dsl-reference)
- [Uploading a Scenario](#uploading-a-scenario)
- [Creating a New Scenario](#creating-a-new-scenario)

## Overview

Each scenario defines:

- **Services** — cloud-native components that emit metrics, logs, and traces with realistic operational patterns
- **Channels** — fault injection definitions (HITL or Auto-Remediate) with error templates, investigation runbooks, root cause clues, and business impact mappings
- **Executive KPIs** — business-level metrics tied to service health
- **An AI Agent** — a configured Elastic AI Assistant persona with domain expertise for the scenario's cloud platform

The platform uses these YAML definitions to generate OpenTelemetry-compatible telemetry and inject faults on demand via the Chaos Console.

## Scenarios

| Scenario | Cloud Provider | Services | Channels | Namespace | Demo Script |
|----------|---------------|----------|----------|-----------|-------------|
| [OVHcloud Network Operations](scenarios/ovh/) | OVHcloud | 6 | 12 (8 HITL + 4 Auto) | `ovhnet` | [EN](scenarios/ovh/script/OVH%20Demo%20Script%20-%20EN.html) / [FR](scenarios/ovh/script/OVH%20Demo%20Script%20-%20FR.html) |

## Directory Structure

```
scenarios/
  <scenario_id>/
    README.md                # Scenario documentation
    scenario.yaml            # Master config: hosts, clusters, agent, KPIs, trace attributes
    services/
      <service-name>.yaml    # Service definition: topology, metrics, logs, state
    channels/
      <NN>-<slug>.yaml       # Fault channel: error template, runbook, RCA clues, business impact
    script/
      *.html / *.md          # Demo scripts for presenters
```

## Schema DSL Reference

All scenarios use **schema version 1.2**. Key constructs:

| Construct | Purpose | Example |
|-----------|---------|---------|
| `sample` | Generate random values | `{randint: [1, 100]}`, `{uniform: [0.0, 1.0], round: 2}`, `{choice: [a, b, c]}` |
| `if_active` / `else` | Conditional on fault injection (inside `sample` only) | `{if_active: {randint: [80, 100]}, else: {randint: [1, 10]}}` |
| `metric` | Emit a metric data point | `{name: lb.rps, value: {var: rps}, unit: req/s}` |
| `log` | Emit a log entry | `{level: INFO, message: '...', attrs: {...}}` |
| `every` | Periodic sub-steps | `{seconds: 10, state_key: _last_check, steps: [...]}` |
| `counter` | Increment state | `{state_key: _poll_idx, by: 1}` |
| `expr` | Computed value | `{expr: "round(used / total * 100, 1)"}` |
| `format` | String interpolation | `{format: "10.0.{a}.{b}", a: {randint: [1, 254]}}` |
| `var` | Reference a sampled variable | `{var: latency_p50}` |

**Important:** `if_active` / `else` works only inside `sample` blocks. Do not use it in `log.level` or `log.message` fields — those expect plain strings.

## Uploading a Scenario

The Chaos Console expects a zip with exactly one top-level directory named after the `scenario_id`:

```bash
cd scenarios/
zip -r ovh.zip ovh/ -x "*.DS_Store"
```

Upload `ovh.zip` via the Chaos Console UI.

## Creating a New Scenario

1. Create `scenarios/<id>/` with `scenario.yaml`, `services/`, and `channels/` directories
2. Set `scenario_id` in `scenario.yaml` to match the directory name
3. Define services with topology, metrics, logs, and `if_active` degradation patterns
4. Define channels with investigation runbooks using real CLI commands, error templates, and business impact mappings
5. Validate: all service names in channels must match service YAML filenames; all `business_impact[].kpi` names must exist in `executive_kpi_emissions`
6. Zip from the `scenarios/` directory and upload
