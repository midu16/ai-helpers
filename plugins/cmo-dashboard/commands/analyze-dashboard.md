---
description: Analyze and validate an existing Grafana dashboard YAML for correctness, metric availability, and diagnostic coverage
argument-hint: "<path-to-dashboard-yaml> [--check-profile minimal|full] [--explain]"
---

## Name
cmo-dashboard:analyze-dashboard

## Synopsis
```
/cmo-dashboard:analyze-dashboard <path-to-dashboard-yaml> [--check-profile minimal|full] [--explain]
```

## Description

The `cmo-dashboard:analyze-dashboard` command reads an existing Grafana dashboard ConfigMap YAML (standalone or bundled) and performs structural validation, metric availability analysis, and diagnostic coverage assessment. It identifies issues that would prevent the dashboard from working correctly in an OpenShift cluster and explains what each panel monitors.

This requires AI reasoning because:
- Validating PromQL semantics (not just syntax) requires understanding what each expression measures
- Determining metric availability requires mapping expressions to collectors and checking default/disabled status
- Assessing diagnostic coverage requires understanding the kernel subsystem or application domain the dashboard targets
- Suggesting improvements requires domain knowledge of what related metrics would complement existing panels

Before analyzing, read the reference skill at `plugins/cmo-dashboard/skills/grafana-dashboard-reference/SKILL.md` for collector details and metric-to-collector mappings.

## Prerequisites

- The dashboard YAML file must be accessible at the provided path
- For `--check-profile minimal`: familiarity with the CMO minimal collection profile allowlist

## Arguments

- `$1` (required): Path to the dashboard YAML file (e.g., `manifests/dashboard-kernel-stack-af-packet-standalone.yaml`)
- `--check-profile` (optional): Check metric availability against a specific collection profile
  - `full` (default): All default-enabled collectors active
  - `minimal`: Only metrics in the minimal collection profile allowlist
- `--explain` (optional): Include a plain-language explanation of what each panel monitors, the investigation workflow, and severity thresholds

## Implementation

### 1. Read and Parse the Dashboard

1. Read the YAML file at the provided path
2. Extract the embedded JSON from the `data` field (the value under the `.json` key)
3. Parse the JSON to access panels, templating, and metadata

### 2. Structural Validation

Check for common structural issues:

| Check | Pass condition | Failure impact |
|---|---|---|
| Valid JSON | No parse errors | Dashboard won't load |
| Unique panel IDs | All `"id"` values distinct | Grafana silently breaks |
| Non-overlapping gridPos | Each panel's `y >= prev.y + prev.h` | Panels overlap visually |
| Datasource reference | `$datasource` matches templating variable | Queries return no data |
| Supported panel types | Only `graph`, `text`, `row`, `singlestat`, `table`, `heatmap` | Panels render as blank |
| Required annotations | `capability.openshift.io/name: Console` present | OCP Console won't discover |
| Required labels | `console.openshift.io/dashboard: "true"` present | OCP Console won't discover |
| Namespace | `openshift-config-managed` | CVO won't apply correctly |

Report each check as PASS, WARN, or FAIL with details.

### 3. Metric Source and Availability Analysis

For each PromQL expression in the dashboard:
1. Classify the metric source:
   - `node_*` → node-exporter (check collector availability)
   - `container_*` → cadvisor (always available; check `container!=""` filter)
   - `kube_*` → kube-state-metrics (always available)
   - `prometheus_*` or `process_*` → Prometheus self-monitoring (always available)
   - Contains `:` → recording rule (check if it exists in CMO rule files)
2. For node-exporter metrics, determine collector availability:

| Status | Meaning |
|---|---|
| Available | Collector is default-enabled and metric is in the collection profile |
| CMO-excluded | Collector is default-enabled but CMO disables it (e.g., `netdev` with `device-exclude=^.*$`) |
| CMO-configurable | Collector is disabled by default but can be enabled via `cluster-monitoring-config` |
| Requires sidecar | Collector cannot be enabled through CMO (`interrupts`, `softirqs`, `zoneinfo`) |
| Not in minimal profile | Metric exists but is excluded from the minimal collection profile allowlist |

3. For recording rules, verify they exist in CMO's recording rule definitions (see Recording Rules section in the skill)
4. For cadvisor metrics, verify proper filters are applied (`container!=""`, `namespace=~".+"`)

Present a summary table of all metrics, their sources, and availability status.

### 4. Collection Profile Check

If `--check-profile minimal` is specified:
1. Check each node-exporter metric against the exact minimal profile allowlist regex from `assets/node-exporter/minimal-service-monitor.yaml` (see the full regex in the skill reference)
2. Note that cadvisor, kube-state-metrics, and Prometheus metrics are not affected by the collection profile
3. Check recording rules — if a recording rule depends on a metric excluded from the minimal profile, it will also return no data
4. Report which panels will show "No data" under the minimal profile
5. Suggest alternatives where possible (e.g., `node_cpu_seconds_total` is in minimal, but `node_softirqs_functions_total` is not)

### 5. Diagnostic Coverage Assessment

Evaluate what the dashboard monitors holistically:
1. Identify the monitoring domain (CPU performance, network, memory, storage, etc.)
2. List which kernel subsystems or application layers are covered
3. Identify gaps — related subsystems that are NOT covered but commonly cause issues in the same domain
4. Suggest additional panels that would improve diagnostic completeness

### 6. Severity Classification (with --explain)

If `--explain` is specified, for each panel provide:
- What the panel measures and why it matters
- Threshold guidance: what values are OK, warning, and critical
- Remediation steps when values are abnormal
- How this panel relates to other panels in the investigation workflow

### 7. Generate Report

Present findings in a structured report:

```
DASHBOARD ANALYSIS: <dashboard-title>
======================================

STRUCTURAL VALIDATION
---------------------
✅ PASS: Valid JSON
✅ PASS: Unique panel IDs (N panels)
❌ FAIL: Overlapping gridPos (panels X and Y)
...

METRIC SOURCES AND AVAILABILITY
-------------------------------
| Metric/Rule | Source | Status |
|-------------|--------|--------|
| node_cpu_seconds_total | node-exporter (cpu) | ✅ Available |
| node_interrupts_total | node-exporter (interrupts) | ⚠️ Requires sidecar |
| container_network_receive_bytes_total | cadvisor | ✅ Available |
| cluster:node_cpu:ratio_rate5m | recording rule | ✅ Defined in CMO |
| namespace_workload_pod:kube_pod_owner:relabel | recording rule | ✅ Defined in CMO |
...

PANELS AFFECTED BY COLLECTION PROFILE
--------------------------------------
(if --check-profile minimal)

DIAGNOSTIC COVERAGE
--------------------
Covers: CPU time breakdown, softirq analysis, IRQ affinity
Gaps: No ethtool stats, no schedstat latency

RECOMMENDATIONS
---------------
1. ...
2. ...
```

## Return Value

- **Format**: Structured analysis report displayed inline
- **Content**: Validation results, metric availability table, coverage assessment, recommendations
- **Exit semantics**:
  - All checks pass → "Dashboard is valid and ready for deployment"
  - Structural issues found → "Dashboard has N issue(s) that must be fixed"
  - Metric availability warnings → "Dashboard is valid but N panel(s) require additional configuration"

## Examples

### Example 1: Validate the AF_PACKET dashboard

```
/cmo-dashboard:analyze-dashboard manifests/dashboard-kernel-stack-af-packet-standalone.yaml
```

Reports structural validation (all pass), metric availability (3 panels require sidecar for `interrupts`/`softirqs`/`zoneinfo`), and notes that `netdev` is CMO-excluded by default.

### Example 2: Check against the minimal collection profile

```
/cmo-dashboard:analyze-dashboard manifests/dashboard-kernel-stack-af-packet-standalone.yaml --check-profile minimal
```

Additionally reports which metrics are excluded from the minimal profile allowlist.

### Example 3: Full analysis with explanations

```
/cmo-dashboard:analyze-dashboard manifests/dashboard-kernel-stack-af-packet-standalone.yaml --explain
```

Includes per-panel explanations with severity thresholds (e.g., "System time % > 5% on isolated CPUs is critical") and the investigation workflow (check system time → softirq → IRQ → conntrack → ...).

### Example 4: Analyze a bundled upstream dashboard

```
/cmo-dashboard:analyze-dashboard manifests/0000_90_cluster-monitoring-operator_01-dashboards.yaml --explain
```

Parses the multi-document YAML (contains 3 dashboards: USE Cluster, USE Node, Prometheus Overview) and analyzes each embedded dashboard separately.

### Example 5: Analyze the compute resources cluster dashboard

```
/cmo-dashboard:analyze-dashboard manifests/0000_90_cluster-monitoring-operator_02-dashboards.yaml --explain
```

Parses the multi-document YAML (contains 9 dashboards) and validates recording rule dependencies, cadvisor metric filters, and kube-state-metrics availability.

## Notes

- Multi-document YAML files (bundled dashboards) are split and each dashboard analyzed independently
- The command does not modify the dashboard file — it is read-only analysis
- For creating new dashboards, use `/cmo-dashboard:create-dashboard`
- PromQL validation is semantic (does the expression make sense?) not syntactic (is it parseable?) — the AI evaluates whether the expression correctly measures what the panel title claims

## See Also

- `plugins/cmo-dashboard/skills/grafana-dashboard-reference/SKILL.md` — collector reference, PromQL patterns, panel templates
- `plugins/cmo-dashboard/commands/create-dashboard.md` — create new dashboards
