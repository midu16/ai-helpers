---
description: Create a standalone Grafana dashboard ConfigMap for the OpenShift cluster-monitoring-operator
argument-hint: "<topic> [--panels <panel-descriptions>] [--source <metric-sources>] [--template-vars <variables>]"
---

## Name
cmo-dashboard:create-dashboard

## Synopsis
```
/cmo-dashboard:create-dashboard <topic> [--panels <panel-descriptions>] [--source <metric-sources>] [--template-vars <variables>]
```

## Description

The `cmo-dashboard:create-dashboard` command generates a complete standalone Grafana dashboard ConfigMap YAML for the OpenShift cluster-monitoring-operator (CMO). The user describes what they want to monitor and the AI selects appropriate metrics from any CMO metric source (node-exporter, cadvisor, kube-state-metrics, Prometheus self-monitoring, or recording rules), chooses PromQL expressions, panel types, template variables, and layout, then produces a valid YAML file ready for inclusion in CMO manifests.

This requires AI reasoning because:
- The user describes a monitoring goal in natural language (e.g., "network saturation", "pod CPU throttling", "Prometheus health", "kernel overhead on isolated CPUs")
- The AI must map that goal to the correct metric source(s) and metrics — this may span multiple sources (e.g., cadvisor for container metrics + kube-state-metrics for resource limits)
- The AI must decide whether to use raw metrics or pre-aggregated recording rules (CMO defines ~120 recording rules; see the skill reference)
- PromQL expressions must be chosen based on the metric type (counter vs gauge), desired visualization (ratio, topk, stacked breakdown, overlay, table), and the appropriate rate function (`rate()` vs `irate()` vs `increase()`)
- Template variables must be selected to match the dashboard's drill-down scope (cluster, namespace, node, pod, workload)
- Panel layout must follow Grafana grid conventions with non-overlapping positions
- Metric availability must be verified against collector defaults, CMO configuration, and the minimal collection profile

Before creating the dashboard, read the reference skill at `plugins/cmo-dashboard/skills/grafana-dashboard-reference/SKILL.md` for metric sources, recording rules, collector details, PromQL patterns, template variable patterns, bundled dashboard catalogue, and panel templates.

## Prerequisites

- Familiarity with the `/apps/cluster-monitoring-operator` repository structure (or the target repo where the dashboard will live)
- Understanding of which metrics the user needs to visualize
- Knowledge of whether any disabled-by-default collectors are required (node-exporter dashboards only)

## Arguments

- `$1` (required): Topic name for the dashboard in kebab-case (e.g., `kernel-stack-af-packet`, `numa-memory`, `pod-cpu-throttling`, `prometheus-health`). Used to derive filenames, ConfigMap name, data key, and UID.
- `--panels` (optional): Comma-separated natural-language descriptions of desired panels. If omitted, the AI infers appropriate panels from the topic.
- `--source` (optional): Comma-separated list of metric sources to use: `node-exporter`, `cadvisor`, `kube-state-metrics`, `prometheus`, `recording-rules`. If omitted, the AI selects sources based on the topic.
- `--template-vars` (optional): Comma-separated list of template variables to include (e.g., `namespace,pod` or `role,instance`). If omitted, the AI selects appropriate variables based on the dashboard scope.

## Implementation

### 1. Read Reference Material

Read `plugins/cmo-dashboard/skills/grafana-dashboard-reference/SKILL.md` to load:
- YAML skeleton template
- Panel type templates (row, text, graph variants)
- Collector reference (default-enabled vs disabled)
- PromQL pattern catalogue
- Naming conventions

### 2. Determine Dashboard Scope and Metric Sources

Based on the topic, `--source`, and `--panels` arguments:

1. Identify which monitoring domain the topic falls under (see Dashboard Topic Selection Guide in the skill):
   - **Node-level**: USE method, kernel stack, NUMA — source: node-exporter / recording rules
   - **Container networking**: Cluster/namespace/pod bandwidth — source: cadvisor `container_network_*`
   - **Compute resources**: CPU/memory usage, requests, limits — source: recording rules + cadvisor + kube-state-metrics
   - **Prometheus health**: Scrape performance, TSDB state — source: Prometheus self-monitoring metrics
   - **Custom/mixed**: May combine multiple sources

2. Select the closest existing bundled dashboard as a starting template (see Bundled Dashboard Catalogue in the skill)

3. Map the topic to specific metrics:
   - Check recording rules first — prefer pre-aggregated rules over raw expressions
   - For node-exporter metrics, verify collector availability:
     - Default-enabled → works out of the box
     - CMO-configurable but disabled → note in output that `cluster-monitoring-config` must enable it
     - Not CMO-configurable (`interrupts`, `softirqs`, `zoneinfo`) → note that a custom sidecar is required
   - For cadvisor metrics, ensure proper filters: `container!=""`, `namespace=~".+"`
   - For kube-state-metrics, use `job="kube-state-metrics"` filter

4. Check metric availability against the minimal collection profile allowlist

### 3. Select Template Variables

Choose template variables based on dashboard scope:

| Scope | Required variables | Pattern |
|---|---|---|
| Cluster-wide | `$datasource` | No selectors needed |
| Per-namespace | `$datasource`, `$namespace` | `label_values(kube_pod_info{}, namespace)` |
| Per-pod | `$datasource`, `$namespace`, `$pod` | Pod query filtered by namespace |
| Per-node | `$datasource`, `$role`, `$instance` | Instance query filtered by role |
| Per-workload | `$datasource`, `$namespace`, `$type`, `$workload` | Workload query via recording rule |
| Network dashboards | Add `$resolution`, `$interval` | Interval selector for `irate()` window |
| Prometheus health | `$datasource`, `$job`, `$instance` | Job query for Prometheus instances |

See Template Variable Patterns in the skill for exact JSON templates.

### 4. Design Panel Layout

For each metric group:
1. Create a `row` panel as section header
2. Optionally add a `text` panel (markdown) explaining the section's purpose
3. Choose the right panel type:

| Data shape | Panel type | When to use |
|---|---|---|
| Time series | `graph` | Most metric visualizations |
| Single current value | `singlestat` | Cluster-level KPIs (utilisation %, commitment %) |
| Multi-column snapshot | `table` | Resource quota tables, status tables |
| Section divider | `row` | Group related panels |
| Explanation | `text` | Markdown descriptions before complex sections |

4. Select PromQL patterns based on metric behaviour:

| Metric behavior | PromQL pattern | Example |
|---|---|---|
| Fractional CPU time | `rate(X{mode="..."}[1m]) / rate(X[1m])` | System time % per CPU |
| High-cardinality counters | `topk(N, rate(X[1m]))` | IRQ rate per CPU |
| Proportional breakdown | `rate(X) by (labels) / rate(X) by (subset)` | Softirq type % per CPU |
| Current value vs ceiling | Two targets on same panel (gauge + limit) | Conntrack entries vs limit |
| Event counts per window | `increase(X[1m])` | Conntrack drops |
| Container network bandwidth | `sum(irate(X[$interval:$resolution])) by (namespace)` | Namespace rx/tx bytes |
| Container network ranked | `sort_desc(sum(irate(X[$interval:$resolution])) by (label))` | Top namespaces by traffic |
| Cluster commitment ratio | `sum(requests) / sum(allocatable)` | CPU/memory commitment % |
| CPU throttling | `rate(throttled_periods[5m]) / rate(total_periods[5m])` | Pod CPU throttling % |
| Pre-aggregated recording rule | `recording_rule_name{labels}` | USE method panels |
| Table with multiple queries | Multiple targets with `"instant": true, "format": "table"` | Resource quota tables |

5. Assign `gridPos` values ensuring no overlaps:
   - Three panels per row: `w: 8` each
   - Two panels per row: `w: 12` each
   - Full-width: `w: 24`
   - Singlestats: `h: 3`, `w: 4` (six per row for cluster KPIs)
   - Each panel's `y` must be >= previous panel's `y + h`

6. Assign unique panel `id` values (start at 100, increment)

### 5. Generate the YAML

Apply naming conventions:

| Element | Convention | Example |
|---|---|---|
| Filename | `dashboard-<topic>-standalone.yaml` | `dashboard-numa-memory-standalone.yaml` |
| ConfigMap name | `dashboard-<topic>` | `dashboard-numa-memory` |
| Data key | `<topic>.json` | `numa-memory.json` |
| Dashboard title | `Category / Name` | `Node Exporter / NUMA Memory Placement` |
| Dashboard UID | `<category>-<topic>` | `node-numa-memory` |

Build the complete YAML:
1. YAML comment header describing the dashboard's purpose
2. `apiVersion: v1` ConfigMap with embedded Grafana JSON
3. All required metadata annotations and labels:
   - `capability.openshift.io/name: Console`
   - `include.release.openshift.io/hypershift: "true"`
   - `include.release.openshift.io/ibm-cloud-managed: "true"`
   - `include.release.openshift.io/self-managed-high-availability: "true"`
   - `include.release.openshift.io/single-node-developer: "true"`
   - `app.kubernetes.io/part-of: openshift-monitoring`
   - `console.openshift.io/dashboard: "true"`
4. `namespace: openshift-config-managed`

### 6. Validate the Output

Before presenting the dashboard, verify:
1. The embedded JSON is valid (no trailing commas, balanced braces)
2. All panel `id` values are unique
3. No `gridPos` overlaps exist
4. `$datasource` matches the templating variable name
5. All metadata annotations and labels are present
6. Only supported panel types are used (`graph`, `text`, `row`, `singlestat`, `table`, `heatmap` — NOT `timeseries`, `stat`, `gauge`, `bargauge`, `piechart`)

### 7. Document Metric Source Requirements

Append a requirements section listing:
- Which metric sources are needed (node-exporter, cadvisor, kube-state-metrics, etc.)
- Which recording rules are used and whether they exist in CMO's rule files
- For node-exporter metrics: whether the collector is default-enabled, CMO-configurable, or requires a custom sidecar
- For cadvisor metrics: note that `container_network_*` metrics are available by default
- Which panels will show "No data" under the minimal collection profile
- Any `cluster-monitoring-config` changes required (e.g., re-enabling `netdev`)

## Return Value

- **Format**: A complete standalone dashboard YAML file written to the working directory or displayed inline
- **Content**: Valid Kubernetes ConfigMap with embedded Grafana JSON
- **Supplementary**: Notes on collector requirements, metric availability, and any caveats

## Examples

### Example 1: Create a NUMA memory dashboard (node-exporter)

```
/cmo-dashboard:create-dashboard numa-memory
```

The AI determines that NUMA monitoring requires:
- `node_zoneinfo_numa_miss_total` and `node_zoneinfo_numa_hit_total` (zoneinfo collector — disabled by default)
- `node_memory_HugePages_*` (meminfo collector — default)
- Panel layout: miss ratio graph, hugepages overview, text explanation

Output: `dashboard-numa-memory-standalone.yaml` with a note that the `zoneinfo` collector requires a custom sidecar.

### Example 2: Create a network saturation dashboard with specific panels

```
/cmo-dashboard:create-dashboard nic-saturation --panels "throughput per device, receive drops and errors, conntrack usage"
```

The AI creates three graph sections using `netdev` and `conntrack` collectors (both default-enabled), with a note that CMO disables `netdev` by default via `--collector.netdev.device-exclude=^.*$`.

### Example 3: Create a pod CPU throttling dashboard (cadvisor + kube-state-metrics)

```
/cmo-dashboard:create-dashboard pod-cpu-throttling --template-vars "namespace,pod"
```

The AI builds a dashboard using `container_cpu_cfs_throttled_periods_total`, `container_cpu_cfs_periods_total` (cadvisor), and `kube_pod_container_resource_limits` (kube-state-metrics). Includes template variables for namespace and pod drill-down. Modeled after the bundled `dashboard-k8s-resources-pod`.

### Example 4: Create a namespace network overview (cadvisor)

```
/cmo-dashboard:create-dashboard namespace-network --source cadvisor --template-vars "namespace,resolution,interval"
```

The AI creates bandwidth, packet rate, and drop panels using `container_network_*` metrics with `irate()` and `[$interval:$resolution]` pattern. Modeled after the bundled `dashboard-namespace-by-pod`.

### Example 5: Create a Prometheus health dashboard (Prometheus self-monitoring)

```
/cmo-dashboard:create-dashboard prometheus-health --source prometheus
```

The AI uses `prometheus_tsdb_head_series`, `prometheus_tsdb_head_samples_appended_total`, `prometheus_engine_query_duration_seconds`, and scrape failure metrics. Modeled after the bundled `dashboard-prometheus`.

### Example 6: Create a cluster resource commitment dashboard (recording rules)

```
/cmo-dashboard:create-dashboard cluster-resource-commitment --source recording-rules --panels "CPU and memory commitment ratios, top namespaces by usage"
```

The AI uses `cluster:node_cpu:ratio_rate5m`, `namespace_cpu:kube_pod_container_resource_requests:sum`, and `kube_node_status_allocatable` to build singlestat KPIs and per-namespace quota tables.

### Example 7: Create a kernel overhead dashboard (node-exporter with disabled collectors)

```
/cmo-dashboard:create-dashboard cpu-isolation --source node-exporter --panels "system time, softirq breakdown, IRQ per device"
```

The AI builds a dashboard focused on CPU isolation monitoring, noting that `softirqs` and `interrupts` require a custom sidecar.

## Notes

- Dashboards are CVO-applied ConfigMaps, not reconciled by Go code
- The OCP Console discovers dashboards via the `console.openshift.io/dashboard: "true"` label
- Default CMO excludes all `netdev`/`netclass` devices; re-enable via `cluster-monitoring-config` if node-level network panels are needed
- cadvisor `container_network_*` metrics are always available (they bypass `netdev` exclusion)
- Prefer recording rules over raw expressions when they exist — they are pre-aggregated and cheaper to query
- CMO defines ~120 recording rules; see the Recording Rules section in the skill reference
- Use `topk()` wrappers for high-cardinality metrics to avoid overwhelming Grafana
- Always include `device!="lo"` filters for node-level network metrics
- Always include `container!=""` filters for cadvisor container metrics
- Use `$__rate_interval` or `[$interval:$resolution]` instead of hardcoded rate windows where appropriate

## See Also

- `plugins/cmo-dashboard/skills/grafana-dashboard-reference/SKILL.md` — collector reference, PromQL patterns, panel templates
- `plugins/cmo-dashboard/commands/analyze-dashboard.md` — analyze and validate existing dashboards
