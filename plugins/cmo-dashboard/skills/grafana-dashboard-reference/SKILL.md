---
name: Grafana Dashboard Reference
description: |
  Reference material for creating and analyzing Grafana dashboard ConfigMaps
  for the OpenShift cluster-monitoring-operator (CMO). Covers all metric sources
  (node-exporter, cadvisor, kube-state-metrics, Prometheus), recording rules,
  panel templates, template variable patterns, and all 12 bundled dashboards.
---

# Grafana Dashboard Reference for CMO

Reference material consumed by the `cmo-dashboard:create-dashboard` and
`cmo-dashboard:analyze-dashboard` commands. Covers all metric sources in
the CMO stack, recording rules, panel templates, template variable patterns,
naming conventions, and all 12 bundled dashboards plus the AF_PACKET standalone
dashboard as worked examples.

## Dashboard YAML Skeleton

All standalone dashboards use this ConfigMap structure:

```yaml
# Dashboard for <TOPIC>.
---
apiVersion: v1
data:
  <data-key>.json: |-
    {
        "annotations": {"list": []}, "editable": true, "gnetId": null,
        "graphTooltip": 1, "hideControls": false, "id": null, "links": [],
        "refresh": "30s", "panels": [],
        "templating": {"list": [
            {"current": {"text": "default", "value": "default"}, "hide": 0,
             "label": "Data Source", "name": "datasource", "query": "prometheus",
             "refresh": 1, "type": "datasource"}
        ]},
        "time": {"from": "now-1h", "to": "now"},
        "timepicker": {"refresh_intervals": ["5s","10s","30s","1m","5m","15m","30m","1h","2h","1d"],
                       "time_options": ["5m","15m","1h","6h","12h","24h","2d","7d","30d"]},
        "timezone": "UTC", "title": "Category / Dashboard Title",
        "uid": "unique-dashboard-slug", "version": 0
    }
kind: ConfigMap
metadata:
  annotations:
    capability.openshift.io/name: Console
    include.release.openshift.io/hypershift: "true"
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    include.release.openshift.io/single-node-developer: "true"
  labels:
    app.kubernetes.io/part-of: openshift-monitoring
    console.openshift.io/dashboard: "true"
  name: dashboard-REPLACE-ME
  namespace: openshift-config-managed
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Filename | `dashboard-<topic>-standalone.yaml` | `dashboard-kernel-stack-af-packet-standalone.yaml` |
| ConfigMap name | `dashboard-<topic>` | `dashboard-kernel-stack-af-packet` |
| Data key | `<topic>.json` | `kernel-stack-af-packet.json` |
| Dashboard title | `Category / Name` | `Node Exporter / Kernel Stack Cost & AF_PACKET` |
| Dashboard UID | `<category>-<topic>` | `node-kernel-stack-af-packet` |

### Dashboard Types

| Type | When to use | Location |
|------|------------|----------|
| **Standalone** | Team-authored, single-purpose dashboard | `manifests/dashboard-<topic>-standalone.yaml` |
| **Bundled** | Upstream-origin, part of a multi-document set | `manifests/0000_90_*_0N-dashboards.yaml` |

For new dashboards, always use **standalone**.

## Panel Templates

Only these panel types are supported by the OCP Console Grafana:

| Supported | NOT supported (Grafana 8+/11) |
|-----------|-------------------------------|
| `graph`, `text`, `row`, `singlestat`, `table`, `heatmap` | `timeseries`, `stat`, `gauge`, `bargauge`, `piechart` |

### Row

```json
{"gridPos": {"h": 1, "w": 24, "x": 0, "y": 0}, "id": 100, "panels": [],
 "showTitle": true, "title": "Section Title", "titleSize": "h6", "type": "row"}
```

### Text (Markdown)

```json
{"content": "**Description** of what follows.", "gridPos": {"h": 2, "w": 24, "x": 0, "y": 1},
 "id": 99, "mode": "markdown", "panels": [], "title": "", "type": "text"}
```

### Graph — Single Target, Ratio Pattern

```json
{"datasource": "$datasource", "gridPos": {"h": 8, "w": 8, "x": 0, "y": 3}, "id": 101,
 "legend": {"show": true, "alignAsTable": true, "current": true, "values": true, "rightSide": true},
 "lines": true, "linewidth": 1,
 "targets": [{"expr": "sum(rate(node_cpu_seconds_total{job=\"node-exporter\", mode=\"system\"}[1m])) by (instance, cpu) / sum(rate(node_cpu_seconds_total{job=\"node-exporter\"}[1m])) by (instance, cpu)",
   "format": "time_series", "legendFormat": "{{instance}} cpu{{cpu}}", "refId": "A"}],
 "title": "System time % per CPU", "type": "graph",
 "yaxes": [{"format": "percentunit", "min": 0, "show": true}, {"show": false}]}
```

### Graph — Stacked Breakdown

```json
{"datasource": "$datasource", "gridPos": {"h": 8, "w": 24, "x": 0, "y": 11}, "id": 132,
 "legend": {"show": true, "alignAsTable": true, "current": true, "values": true},
 "lines": true, "stack": true,
 "targets": [{"expr": "sum(rate(node_softirqs_functions_total{job=~\"node-exporter.*\"}[1m])) by (instance, cpu, type) / sum(rate(node_softirqs_functions_total{job=~\"node-exporter.*\"}[1m])) by (instance, cpu)",
   "format": "time_series", "legendFormat": "{{instance}} cpu{{cpu}} {{type}}", "refId": "A"}],
 "title": "Per CPU: softirq type as % of invocations", "type": "graph",
 "yaxes": [{"format": "percentunit", "min": 0, "max": 1, "show": true}, {"show": false}]}
```

### Graph — topk, High-Cardinality Filter

```json
{"datasource": "$datasource", "gridPos": {"h": 8, "w": 24, "x": 0, "y": 19}, "id": 135,
 "legend": {"show": true, "alignAsTable": true, "current": true, "values": true},
 "lines": true, "stack": true,
 "targets": [{"expr": "topk(100, rate(node_interrupts_total{devices=~\".*TxRx.*|.*eno.*|.*net.*\"}[1m]))",
   "format": "time_series", "legendFormat": "{{instance}} cpu{{cpu}} {{devices}}", "refId": "A"}],
 "title": "IRQ rate per CPU (NIC queues only)", "type": "graph",
 "yaxes": [{"format": "ops", "min": 0, "show": true}, {"show": false}]}
```

### Graph — Multi-Target Overlay

```json
{"datasource": "$datasource", "gridPos": {"h": 8, "w": 12, "x": 0, "y": 27}, "id": 109,
 "legend": {"show": true, "alignAsTable": true, "current": true, "values": true},
 "lines": true,
 "targets": [
   {"expr": "node_nf_conntrack_entries{job=\"node-exporter\"}", "format": "time_series",
    "legendFormat": "{{instance}} entries", "refId": "A"},
   {"expr": "node_nf_conntrack_entries_limit{job=\"node-exporter\"}", "format": "time_series",
    "legendFormat": "{{instance}} limit", "refId": "B"}],
 "title": "Conntrack entries vs limit", "type": "graph",
 "yaxes": [{"format": "short", "min": 0, "show": true}, {"show": false}]}
```

### Graph — Increase Pattern, Event Counting

```json
{"datasource": "$datasource", "gridPos": {"h": 8, "w": 12, "x": 0, "y": 43}, "id": 110,
 "legend": {"show": true, "alignAsTable": true, "current": true, "values": true}, "lines": true,
 "targets": [
   {"expr": "increase(node_nf_conntrack_stat_drop{job=\"node-exporter\"}[1m])",
    "format": "time_series", "legendFormat": "{{instance}} drop", "refId": "A"},
   {"expr": "increase(node_nf_conntrack_stat_early_drop{job=\"node-exporter\"}[1m])",
    "format": "time_series", "legendFormat": "{{instance}} early_drop", "refId": "B"}],
 "title": "Conntrack drops (increase 1m)", "type": "graph",
 "yaxes": [{"format": "short", "min": 0, "show": true}, {"show": false}]}
```

### Y-Axis Formats

| `"format"` | Meaning |
|-------------|---------|
| `"percentunit"` | 0-1 as percentage |
| `"ops"` | Operations/s |
| `"Bps"` | Bytes/s |
| `"bytes"` | Absolute bytes |
| `"short"` | Plain number |
| `"percent"` | 0-100 as percentage |
| `"decbytes"` | Decimal bytes (auto-scale KiB/MiB/GiB) |
| `"pps"` | Packets/s |
| `"iops"` | I/O operations/s |

### Singlestat

Used in k8s-resources dashboards for cluster-wide commitment ratios:

```json
{"datasource": "$datasource", "gridPos": {"h": 3, "w": 4, "x": 0, "y": 0}, "id": 1,
 "format": "percentunit", "maxValue": 100, "sparkline": {"show": true},
 "targets": [{"expr": "cluster:node_cpu:ratio_rate5m{}", "refId": "A"}],
 "thresholds": "70,80", "title": "CPU Utilisation", "type": "singlestat",
 "valueFontSize": "80%", "valueName": "current"}
```

### Table — Multi-Column with Overrides

Used for resource quota summaries (CPU Quota, Memory Quota, Network Status):

```json
{"datasource": "$datasource", "gridPos": {"h": 12, "w": 24, "x": 0, "y": 11}, "id": 8,
 "styles": [
   {"alias": "Pods", "pattern": "Value #A", "type": "number", "decimals": 0},
   {"alias": "CPU Usage", "pattern": "Value #B", "type": "number", "decimals": 3},
   {"alias": "CPU Requests", "pattern": "Value #C", "type": "number", "decimals": 3},
   {"alias": "CPU Requests %", "pattern": "Value #D", "type": "number", "unit": "percentunit", "decimals": 1}],
 "targets": [
   {"expr": "count(namespace_workload_pod:kube_pod_owner:relabel{namespace=\"$namespace\"}) by (workload, workload_type)",
    "format": "table", "instant": true, "refId": "A"},
   {"expr": "sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace=\"$namespace\"}) by (workload)",
    "format": "table", "instant": true, "refId": "B"}],
 "title": "CPU Quota", "transform": "table", "type": "table"}
```

### Layout Rules

- Three panels per row: `w: 8` each — use for per-CPU metrics
- Two panels per row: `w: 12` each — use for comparison panels
- Full-width: `w: 24` — use for stacked breakdowns and high-cardinality graphs
- Each panel's `y` must be >= previous panel's `y + h`
- Row panels have `h: 1`, text panels typically `h: 2`, graph panels typically `h: 8`

## Metric Availability Checklist

Before adding a metric to a panel, verify:

1. Is the collector **default-enabled** or **disabled-by-default**?
2. If default-enabled, is the metric in the **minimal collection profile** allowlist?
3. Does CMO disable the collector by default (e.g., `netdev` is excluded with `^.*$`)?

See the Node-Exporter Collector Reference and Collection Profiles sections below.

## Dashboard Validation Checklist

1. Valid JSON (embedded in YAML `|-` block)
2. Unique panel `"id"` values
3. No overlapping `gridPos.y` values
4. `$datasource` matches templating variable name
5. All metadata annotations and labels present
6. Metrics available in the required collection profile

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| Invalid JSON in YAML | Malformed JSON inside `|-` block | `python3 -c "import json; json.load(open('file.json'))"` to validate |
| Missing metrics / "No data" | Collector not enabled | Check Collector Reference (Appendix A) — enable via `cluster-monitoring-config` or custom sidecar |
| No network metrics | CMO sets `--collector.netdev.device-exclude=^.*$` and `--collector.netclass.ignored-devices=^.*$` by default | Re-enable via `cluster-monitoring-config` |
| No softirq/interrupt/zoneinfo data | Collectors disabled by default, not CMO-configurable | Deploy custom sidecar node-exporter with `--collector.interrupts --collector.softirqs --collector.zoneinfo` |
| Collection profile mismatch | Metric not in the `minimal` profile allowlist | Check `assets/node-exporter/minimal-service-monitor.yaml` regex |

## Output Format

A completed standalone dashboard file follows this structure:

```
manifests/dashboard-<topic>-standalone.yaml
├── YAML comment header (purpose of dashboard)
├── apiVersion: v1
├── data:
│   └── <topic>.json: |-
│       └── Grafana JSON object
│           ├── annotations, editable, graphTooltip, refresh
│           ├── panels: [...]
│           │   ├── text panels (markdown descriptions)
│           │   ├── row panels (section headers)
│           │   └── graph panels (PromQL targets)
│           ├── templating: datasource variable
│           ├── time: default window
│           └── title, uid, version
├── kind: ConfigMap
└── metadata:
    ├── annotations (capability, release includes)
    ├── labels (app.kubernetes.io/part-of, console.openshift.io/dashboard)
    ├── name: dashboard-<topic>
    └── namespace: openshift-config-managed
```

## PromQL Pattern Catalogue

| Pattern | Template | When to use |
|---------|---------|-------------|
| **Ratio (rate/rate)** | `rate(X{mode="..."}[1m]) / rate(X[1m])` | Fractional CPU time per mode |
| **topk(N, rate())** | `topk(100, rate(X[1m]))` | High-cardinality metrics; show only hottest series |
| **Stacked ratio** | `rate(X) by (cpu,type) / rate(X) by (cpu)` | Proportions summing to 100% |
| **Gauge overlay** | `X_entries` + `X_limit` on same panel | Current value vs ceiling |
| **increase()** | `increase(X[1m])` | Event counts per window |
| **irate() by namespace** | `sum(irate(X{namespace=~".+"}[$interval:$resolution])) by (namespace)` | Container network bandwidth |
| **sort_desc(sum())** | `sort_desc(sum(irate(X[$interval:$resolution])) by (label))` | Ranked namespace/pod bandwidth |
| **Recording rule** | `recording_rule_name{labels}` | Pre-aggregated metrics (see Recording Rules section) |
| **Commitment ratio** | `sum(requests) / sum(allocatable)` | Cluster resource commitment % |
| **Instant table** | `expr` with `"instant": true, "format": "table"` | Snapshot tables (CPU Quota, Memory Quota) |

### Pattern Selection Guide

- **Counter metrics** (name ends in `_total`): Always wrap in `rate()` or `increase()`
- **Gauge metrics**: Use directly, no wrapping needed
- **Per-CPU breakdown**: Use `by (instance, cpu)` in aggregation
- **Per-namespace breakdown**: Use `by (namespace)` in aggregation
- **Per-pod breakdown**: Use `by (pod)` in aggregation, filter with `namespace="$namespace"`
- **Per-workload breakdown**: Use recording rule `namespace_workload_pod:kube_pod_owner:relabel` to join workload labels
- **Ratios**: Divide filtered rate by unfiltered rate for the same metric
- **Filtering high cardinality**: Use `topk(N, ...)` or label regex filters
- **Rate interval**: Use `$__rate_interval` in dashboards with Grafana auto-interval; use `[$interval:$resolution]` when explicit interval/resolution template variables are defined
- **Table panels**: Use `"instant": true` and `"format": "table"` with `"transform": "table"` for snapshot data

## Template Variable Patterns

Dashboards use Grafana template variables so users can drill down. These patterns
are extracted from the 12 bundled CMO dashboards.

### Datasource (required in all dashboards)

```json
{"current": {"text": "default", "value": "default"}, "hide": 0,
 "label": "Data Source", "name": "datasource", "query": "prometheus",
 "refresh": 1, "type": "datasource"}
```

### Namespace Selector

```json
{"datasource": "$datasource", "hide": 0, "includeAll": false,
 "label": "namespace", "name": "namespace",
 "query": "label_values(kube_pod_info{}, namespace)",
 "refresh": 2, "sort": 1, "type": "query"}
```

### Pod Selector (filtered by namespace)

```json
{"datasource": "$datasource", "hide": 0, "includeAll": false,
 "label": "pod", "name": "pod",
 "query": "label_values(kube_pod_info{namespace=\"$namespace\"}, pod)",
 "refresh": 2, "sort": 1, "type": "query"}
```

### Node / Instance Selector (filtered by role)

```json
{"datasource": "$datasource", "hide": 0, "includeAll": false,
 "label": "instance", "name": "instance",
 "query": "label_values(kube_node_info{} * on (node) group_right kube_node_role{role=~\"$role\"}, node)",
 "refresh": 2, "sort": 1, "type": "query"}
```

### Role Selector

```json
{"datasource": "$datasource", "hide": 0, "includeAll": true,
 "label": "role", "name": "role",
 "query": "label_values(kube_node_role{}, role)",
 "refresh": 2, "sort": 1, "type": "query"}
```

### Workload Type Selector

```json
{"datasource": "$datasource", "hide": 0, "includeAll": true,
 "label": "workload_type", "name": "type",
 "query": "label_values(namespace_workload_pod:kube_pod_owner:relabel{namespace=\"$namespace\", workload=~\".+\"}, workload_type)",
 "refresh": 2, "sort": 1, "type": "query"}
```

### Resolution / Interval (network dashboards)

```json
{"auto": false, "auto_count": 30, "auto_min": "10s",
 "current": {"text": "5m", "value": "5m"}, "hide": 0,
 "label": "Resolution", "name": "resolution",
 "options": [{"text": "30s", "value": "30s"}, {"text": "5m", "value": "5m"}, {"text": "1h", "value": "1h"}],
 "query": "30s,5m,1h", "refresh": 2, "type": "interval"}
```

### Prometheus Job/Instance Selector

```json
{"datasource": "$datasource", "hide": 0, "includeAll": false,
 "label": "job", "name": "job",
 "query": "label_values(prometheus_build_info{job=~\"prometheus-k8s|prometheus-user-workload\"}, job)",
 "refresh": 2, "sort": 1, "type": "query"}
```

## Metric Sources Beyond Node-Exporter

CMO dashboards use four primary metric sources. Each source has a different
job label and provides different metric families.

### cadvisor — Container Resource Metrics

Source: kubelet `/metrics/cadvisor` endpoint.
Job label: `job="kubelet"`, `metrics_path="/metrics/cadvisor"`.

| Metric | Type | Labels | Used in |
|--------|------|--------|---------|
| `container_cpu_usage_seconds_total` | counter | `namespace`, `pod`, `container` | Compute Resources dashboards |
| `container_cpu_cfs_periods_total` | counter | `namespace`, `pod`, `container` | CPU Throttling panel |
| `container_cpu_cfs_throttled_periods_total` | counter | `namespace`, `pod`, `container` | CPU Throttling panel |
| `container_memory_working_set_bytes` | gauge | `namespace`, `pod`, `container` | Memory Usage panels |
| `container_memory_rss` | gauge | `namespace`, `pod`, `container` | Memory Usage (w/o cache) |
| `container_memory_cache` | gauge | `namespace`, `pod`, `container` | Memory Quota tables |
| `container_memory_swap` | gauge | `namespace`, `pod`, `container` | Memory Quota tables |
| `container_network_receive_bytes_total` | counter | `namespace`, `pod` | All Networking dashboards |
| `container_network_transmit_bytes_total` | counter | `namespace`, `pod` | All Networking dashboards |
| `container_network_receive_packets_total` | counter | `namespace`, `pod` | Packet rate panels |
| `container_network_transmit_packets_total` | counter | `namespace`, `pod` | Packet rate panels |
| `container_network_receive_packets_dropped_total` | counter | `namespace`, `pod` | Dropped packets panels |
| `container_network_transmit_packets_dropped_total` | counter | `namespace`, `pod` | Dropped packets panels |
| `container_fs_reads_total` | counter | `namespace`, `pod`, `device` | IOPS panels |
| `container_fs_writes_total` | counter | `namespace`, `pod`, `device` | IOPS panels |
| `container_fs_reads_bytes_total` | counter | `namespace`, `pod`, `device` | Throughput panels |
| `container_fs_writes_bytes_total` | counter | `namespace`, `pod`, `device` | Throughput panels |

**Important filters**: Always include `container!=""` to exclude pod-level aggregations.
For network metrics, use `namespace=~".+"` to exclude host-network pods.

### kube-state-metrics — Kubernetes Object State

Source: kube-state-metrics `/metrics` endpoint.
Job label: `job="kube-state-metrics"`.

| Metric | Type | Labels | Used in |
|--------|------|--------|---------|
| `kube_pod_info` | gauge | `namespace`, `pod`, `node` | Template variable queries |
| `kube_pod_owner` | gauge | `namespace`, `pod`, `owner_kind`, `owner_name` | Workload grouping |
| `kube_pod_container_resource_requests` | gauge | `namespace`, `pod`, `container`, `resource` | CPU/Memory Quota tables |
| `kube_pod_container_resource_limits` | gauge | `namespace`, `pod`, `container`, `resource` | CPU/Memory Quota tables |
| `kube_node_status_allocatable` | gauge | `node`, `resource` | Cluster commitment ratios |
| `kube_node_status_capacity` | gauge | `node`, `resource` | Node resource tables |
| `kube_node_role` | gauge | `node`, `role` | Node/instance template variables |
| `kube_node_info` | gauge | `node` | Node template variables |
| `kube_resourcequota` | gauge | `namespace`, `resource`, `type` | Quota panels |

### Prometheus Self-Monitoring

Source: Prometheus `/metrics` endpoint.
Job label: `job=~"prometheus-k8s|prometheus-user-workload"`.

| Metric | Type | Labels | Used in |
|--------|------|--------|---------|
| `prometheus_build_info` | gauge | `job`, `instance`, `version` | Prometheus Overview stats table |
| `prometheus_tsdb_head_series` | gauge | `job`, `instance` | Head Series panel |
| `prometheus_tsdb_head_chunks` | gauge | `job`, `instance` | Head Chunks panel |
| `prometheus_tsdb_head_samples_appended_total` | counter | `job`, `instance` | Appended Samples panel |
| `prometheus_engine_query_duration_seconds` | summary | `job`, `instance`, `slice` | Query Rate / Stage Duration |
| `prometheus_target_sync_length_seconds_sum` | counter | `job`, `instance`, `scrape_job` | Target Sync panel |
| `prometheus_sd_discovered_targets` | gauge | `job`, `instance` | Targets panel |
| `prometheus_target_interval_length_seconds_sum` | counter | `job`, `instance`, `interval` | Scrape Interval panel |
| `prometheus_target_scrapes_exceeded_sample_limit_total` | counter | `job` | Scrape failures panel |
| `prometheus_target_scrapes_sample_out_of_order_total` | counter | `job` | Scrape failures panel |
| `process_start_time_seconds` | gauge | `job`, `instance` | Uptime calculation |

## Recording Rules Used by Dashboards

CMO defines ~120 recording rules. The following are directly consumed by the
bundled dashboards. When creating dashboards, prefer recording rules over raw
expressions when they exist — they are pre-aggregated and cheaper to query.

### Node-Level Recording Rules (USE Method Dashboards)

| Recording Rule | Raw Expression | Used in |
|---------------|----------------|---------|
| `instance:node_cpu_utilisation:rate1m` | `1 - avg without(mode)(rate(node_cpu_seconds_total{mode="idle"}[1m]))` | USE Cluster/Node: CPU Utilisation |
| `instance:node_load1_per_cpu:ratio` | `node_load1 / instance:node_num_cpu:sum` | USE Cluster/Node: CPU Saturation |
| `instance:node_num_cpu:sum` | `count without(cpu,mode)(node_cpu_seconds_total{mode="idle"})` | USE Cluster: CPU normalisation |
| `instance:node_memory_utilisation:ratio` | `1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` | USE Cluster/Node: Memory Utilisation |
| `instance:node_vmstat_pgmajfault:rate1m` | `rate(node_vmstat_pgmajfault[1m])` | USE Cluster/Node: Memory Saturation |
| `instance:node_network_receive_bytes_excluding_lo:rate1m` | `rate(node_network_receive_bytes_total{device!="lo"}[1m])` | USE Cluster/Node: Network Utilisation |
| `instance:node_network_transmit_bytes_excluding_lo:rate1m` | `rate(node_network_transmit_bytes_total{device!="lo"}[1m])` | USE Cluster/Node: Network Utilisation |
| `instance:node_network_receive_drop_excluding_lo:rate1m` | `rate(node_network_receive_drop_total{device!="lo"}[1m])` | USE Cluster/Node: Network Saturation |
| `instance:node_network_transmit_drop_excluding_lo:rate1m` | `rate(node_network_transmit_drop_total{device!="lo"}[1m])` | USE Cluster/Node: Network Saturation |
| `instance_device:node_disk_io_time_seconds:rate1m` | `rate(node_disk_io_time_seconds_total[1m])` | USE Cluster/Node: Disk IO Utilisation |
| `instance_device:node_disk_io_time_weighted_seconds:rate1m` | `rate(node_disk_io_time_weighted_seconds_total[1m])` | USE Cluster/Node: Disk IO Saturation |

### Cluster-Level Recording Rules (Compute Resources Dashboards)

| Recording Rule | Raw Expression | Used in |
|---------------|----------------|---------|
| `cluster:node_cpu:ratio_rate5m` | `sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate) / sum(kube_node_status_allocatable{resource="cpu"})` | Cluster CPU Utilisation singlestat |
| `:node_memory_MemAvailable_bytes:sum` | `sum(node_memory_MemAvailable_bytes) / sum(node_memory_MemTotal_bytes)` | Cluster Memory Utilisation singlestat |
| `namespace_cpu:kube_pod_container_resource_requests:sum` | `sum by(namespace)(kube_pod_container_resource_requests{resource="cpu"})` | CPU Requests Commitment |
| `namespace_cpu:kube_pod_container_resource_limits:sum` | `sum by(namespace)(kube_pod_container_resource_limits{resource="cpu"})` | CPU Limits Commitment |
| `namespace_memory:kube_pod_container_resource_requests:sum` | `sum by(namespace)(kube_pod_container_resource_requests{resource="memory"})` | Memory Requests Commitment |
| `namespace_memory:kube_pod_container_resource_limits:sum` | `sum by(namespace)(kube_pod_container_resource_limits{resource="memory"})` | Memory Limits Commitment |

### Container-Level Recording Rules (Resource Dashboards)

| Recording Rule | Raw Expression | Used in |
|---------------|----------------|---------|
| `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate` | `sum by(cluster,namespace,pod,container)(irate(container_cpu_usage_seconds_total{image!=""}[5m])) * on(cluster,namespace,pod) group_left(node) topk by(cluster,namespace,pod)(1,max by(cluster,namespace,pod,node)(kube_pod_info{}))` | CPU Usage graphs |
| `node_namespace_pod_container:container_memory_working_set_bytes` | Working set bytes per container | Memory Usage graphs |
| `node_namespace_pod_container:container_memory_rss` | RSS per container | Memory Usage (w/o cache) |
| `node_namespace_pod_container:container_memory_cache` | Cache per container | Memory Quota tables |
| `node_namespace_pod_container:container_memory_swap` | Swap per container | Memory Quota tables |
| `cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests` | Active CPU requests | Namespace/Pod CPU tables |
| `cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits` | Active CPU limits | Namespace/Pod CPU tables |
| `cluster:namespace:pod_memory:active:kube_pod_container_resource_requests` | Active memory requests | Namespace/Pod Memory tables |
| `cluster:namespace:pod_memory:active:kube_pod_container_resource_limits` | Active memory limits | Namespace/Pod Memory tables |
| `namespace_workload_pod:kube_pod_owner:relabel` | Pod-to-workload mapping | Workload dashboards |

## Node-Exporter Collector Reference (v1.9.1)

### How Node-Exporter Runs in OpenShift

CMO deploys node-exporter as a `DaemonSet` in `openshift-monitoring` with `hostNetwork: true`,
`hostPID: true`, mounting host fs at `/host/root` and sysfs at `/host/sys`. Default args:

```
--web.listen-address=127.0.0.1:9101  --path.sysfs=/host/sys  --path.rootfs=/host/root
--path.procfs=/host/root/proc  --no-collector.wifi  --no-collector.btrfs
--collector.cpu.info  --collector.textfile.directory=/var/node_exporter/textfile
--collector.netclass.ignored-devices=^.*$  --collector.netdev.device-exclude=^.*$
```

`updateNodeExporterArgs()` in `pkg/manifests/manifests.go` dynamically adjusts flags from
`ClusterMonitoringConfiguration.nodeExporter.collectors`.

### Default-Enabled Collectors

#### cpu — `/proc/stat`, `/proc/cpuinfo`, `/sys/devices/system/cpu/`

| Metric | Type | Labels |
|--------|------|--------|
| `node_cpu_seconds_total` | counter | `cpu`, `mode` (user/nice/system/idle/iowait/irq/softirq/steal) |
| `node_cpu_guest_seconds_total` | counter | `cpu`, `mode` (user/nice) |
| `node_cpu_info` | gauge | `package`, `core`, `cpu`, `vendor`, `family`, `model`, `model_name`, `microcode`, `stepping`, `cachesize` |
| `node_cpu_core_throttles_total` | counter | `package`, `core` |
| `node_cpu_package_throttles_total` | counter | `package` |
| `node_cpu_isolated` | gauge | `cpu` |
| `node_cpu_online` | gauge | `cpu` |
| `node_cpu_frequency_hertz` | gauge | `package`, `core`, `cpu` |

#### pressure — `/proc/pressure/{cpu,io,memory,irq}`

| Metric | Type |
|--------|------|
| `node_pressure_cpu_waiting_seconds_total` | counter |
| `node_pressure_io_waiting_seconds_total` | counter |
| `node_pressure_io_stalled_seconds_total` | counter |
| `node_pressure_memory_waiting_seconds_total` | counter |
| `node_pressure_memory_stalled_seconds_total` | counter |
| `node_pressure_irq_stalled_seconds_total` | counter |

`some` = at least one task stalled (`waiting`); `full` = all non-idle tasks stalled (`stalled`).
Kernel 4.20+ (IRQ: 6.1+).

#### conntrack — `/proc/sys/net/netfilter/`, `/proc/net/stat/nf_conntrack`

| Metric | Type |
|--------|------|
| `node_nf_conntrack_entries` | gauge |
| `node_nf_conntrack_entries_limit` | gauge |
| `node_nf_conntrack_stat_drop` | gauge |
| `node_nf_conntrack_stat_early_drop` | gauge |
| `node_nf_conntrack_stat_found` | gauge |
| `node_nf_conntrack_stat_invalid` | gauge |
| `node_nf_conntrack_stat_insert_failed` | gauge |
| `node_nf_conntrack_stat_search_restart` | gauge |

Returns `ErrNoData` if `nf_conntrack` module not loaded.

#### netstat — `/proc/net/netstat`, `/proc/net/snmp`, `/proc/net/snmp6`

Dynamically generated as `node_netstat_<Protocol>_<Field>` (untyped, no labels).
Default regex: `^(.*_(InErrors|InErrs)|Ip_Forwarding|Ip(6|Ext)_(InOctets|OutOctets)|Icmp6?_(InMsgs|OutMsgs)|TcpExt_(Listen.*|Syncookies.*|TCPSynRetrans|TCPTimeouts|TCPOFOQueue|TCPRcvQDrop)|Tcp_(ActiveOpens|InSegs|OutSegs|OutRsts|PassiveOpens|RetransSegs|CurrEstab)|Udp6?_(InDatagrams|OutDatagrams|NoPorts|RcvbufErrors|SndbufErrors))$`

#### sockstat — `/proc/net/sockstat`, `/proc/net/sockstat6`

| Metric | Type |
|--------|------|
| `node_sockstat_sockets_used` | gauge |
| `node_sockstat_TCP_inuse`, `_orphan`, `_tw`, `_alloc`, `_mem`, `_mem_bytes` | gauge |
| `node_sockstat_UDP_inuse`, `_mem`, `_mem_bytes` | gauge |

`*_mem_bytes` = `*_mem` × page size (typically 4096).

#### softnet — `/proc/net/softnet_stat`

| Metric | Type | Labels |
|--------|------|--------|
| `node_softnet_processed_total` | counter | `cpu` |
| `node_softnet_dropped_total` | counter | `cpu` |
| `node_softnet_times_squeezed_total` | counter | `cpu` |
| `node_softnet_backlog_len` | gauge | `cpu` |

#### stat — `/proc/stat`

| Metric | Type |
|--------|------|
| `node_intr_total` | counter |
| `node_context_switches_total` | counter |
| `node_forks_total` | counter |
| `node_boot_time_seconds` | gauge |
| `node_procs_running` | gauge |
| `node_procs_blocked` | gauge |

#### meminfo — `/proc/meminfo`

~50 gauge metrics. Fields with `kB` → `_bytes` (×1024). `HugePages_*` stay as page counts.
Includes `MemTotal_bytes`, `MemAvailable_bytes`, `HugePages_Total`, `HugePages_Free`,
`HugePages_Rsvd`, `Hugepagesize_bytes`.

#### netdev — `/proc/net/dev` or netlink

| Metric | Type | Labels |
|--------|------|--------|
| `node_network_receive_bytes_total` | counter | `device` |
| `node_network_receive_drop_total` | counter | `device` |
| `node_network_receive_errs_total` | counter | `device` |
| `node_network_transmit_bytes_total` | counter | `device` |
| (plus packets, fifo, frame, colls, carrier, compressed) | counter | `device` |

**CMO default**: `--collector.netdev.device-exclude=^.*$` disables all devices.
Re-enable via `cluster-monitoring-config`.

#### Other Default Collectors

| Collector | Source | Key metrics |
|-----------|--------|-------------|
| arp | `/proc/net/arp` | `node_arp_entries` |
| diskstats | `/proc/diskstats` | `node_disk_{reads,writes}_completed_total`, `io_time_*` |
| entropy | `/proc/sys/kernel/random/` | `node_entropy_available_bits` |
| filefd | `/proc/sys/fs/file-nr` | `node_filefd_allocated`, `_maximum` |
| loadavg | `/proc/loadavg` | `node_load1`, `_5`, `_15` |
| netclass | `/sys/class/net/` | `node_network_info`, `_up`, `_speed_bytes` |
| schedstat | `/proc/schedstat` | `node_schedstat_running_seconds_total` |
| uname | `uname(2)` | `node_uname_info` |
| vmstat | `/proc/vmstat` | `node_vmstat_pgmajfault`, `oom_kill` |

### Disabled-by-Default Collectors

#### interrupts — `/proc/interrupts`

| Metric | Type | Labels |
|--------|------|--------|
| `node_interrupts_total` | counter | `cpu`, `type`, `info`, `devices` |

**Cardinality**: IRQ_lines × CPUs. Filter: `--collector.interrupts.name-include/exclude`.
**Not CMO-configurable.** Requires custom sidecar.

#### softirqs — `/proc/softirqs`

| Metric | Type | Labels |
|--------|------|--------|
| `node_softirqs_functions_total` | counter | `cpu`, `type` |

10 fixed types: `HI`, `TIMER`, `NET_TX`, `NET_RX`, `BLOCK`, `IRQ_POLL`, `TASKLET`, `SCHED`, `HRTIMER`, `RCU`.
Cardinality: 10 × N CPUs. **Not CMO-configurable.** Requires custom sidecar.

#### zoneinfo — `/proc/zoneinfo`

Labels: `node` (NUMA node), `zone` (DMA/DMA32/Normal).
Gauges: `nr_free_pages`, `min_pages`, etc.
Counters: `numa_hit_total`, `numa_miss_total`, `numa_foreign_total`, `numa_local_total`, `numa_other_total`.
**Not CMO-configurable.** Requires custom sidecar.

#### ethtool — ioctl SIOCETHTOOL

`node_ethtool_*` (received/transmitted bytes, packets, errors) + all driver-specific stats.
200-800+ counters per NIC. Filter: `--collector.ethtool.metrics-include`.
**CMO-configurable**: `nodeExporter.collectors.ethtool.enabled`.

#### Other Disabled Collectors

| Collector | Key metric | CMO toggle |
|-----------|------------|------------|
| tcpstat | `node_tcp_connection_states` | `tcpstat.enabled` |
| cpufreq | `node_cpu_frequency_*_hertz` | `cpufreq.enabled` |
| buddyinfo | `node_buddyinfo_blocks` | `buddyinfo.enabled` |
| processes | `node_processes_state` | `processes.enabled` |
| ksmd | `node_ksmd_pages_shared` | `ksmd.enabled` |
| systemd | `node_systemd_unit_state` | `systemd.enabled` |
| mountstats | `node_mountstats_nfs_*` | `mountstats.enabled` |

### Custom Sidecar for Non-CMO-Configurable Collectors

```yaml
image: quay.io/prometheus/node-exporter:v1.9.1
args:
  - --web.listen-address=0.0.0.0:9101
  - --path.procfs=/host/proc
  - --collector.disable-defaults
  - --collector.zoneinfo
  - --collector.interrupts
  - --collector.softirqs
```

Deploy as a sidecar DaemonSet with its own ServiceMonitor.

## CMO Architecture

### Delivery Pipeline

```
manifests/                  Dockerfile                  CVO                         OCP Console
 0000_90_*-dashboards.yaml  -> COPY manifests           -> applies ConfigMaps       -> reads ConfigMaps
 dashboard-*-standalone.yaml   /manifests                  to openshift-config-         with label
                               COPY assets /assets         managed namespace            console.openshift.io/
                                                                                        dashboard: "true"
```

Dashboards are CVO-applied ConfigMaps, **not** reconciled by Go code.

### Collection Profiles

The `minimal` profile (`assets/node-exporter/minimal-service-monitor.yaml`, v1.10.2) uses a
`keep` relabeling with this exact regex on `__name__`:

```
node_cpu_info|node_cpu_seconds_total|node_disk_io_time_seconds_total|
node_disk_io_time_weighted_seconds_total|node_disk_read_time_seconds_total|
node_disk_reads_completed_total|node_disk_write_time_seconds_total|
node_disk_writes_completed_total|node_filefd_allocated|node_filefd_maximum|
node_filesystem_avail_bytes|node_filesystem_files|node_filesystem_files_free|
node_filesystem_free_bytes|node_filesystem_readonly|node_filesystem_size_bytes|
node_load1|node_memory_Buffers_bytes|node_memory_Cached_bytes|
node_memory_MemAvailable_bytes|node_memory_MemFree_bytes|node_memory_MemTotal_bytes|
node_memory_Slab_bytes|node_netstat_TcpExt_TCPSynRetrans|node_netstat_Tcp_OutSegs|
node_netstat_Tcp_RetransSegs|node_network_receive_bytes_total|
node_network_receive_drop_total|node_network_receive_errs_total|
node_network_receive_packets_total|node_network_transmit_bytes_total|
node_network_transmit_drop_total|node_network_transmit_errs_total|
node_network_transmit_packets_total|node_network_up|node_nf_conntrack_entries|
node_nf_conntrack_entries_limit|node_textfile_scrape_error|
node_timex_maxerror_seconds|node_timex_offset_seconds|node_timex_sync_status|
node_vmstat_pgmajfault|process_start_time_seconds|virt_platform
```

Metrics **NOT** in the minimal profile but used by dashboards:
`node_memory_HugePages_*`, `node_pressure_*`, `node_sockstat_*`, `node_softnet_*`,
`node_nf_conntrack_stat_*`, `node_netstat_TcpExt_TCPRcvQDrop`,
`node_netstat_TcpExt_Listen*`, `node_netstat_Udp_*`.

### CMO-Configurable Collectors

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    nodeExporter:
      collectors:
        cpufreq:    { enabled: true }
        tcpstat:    { enabled: true }
        netdev:     { enabled: true }
        netclass:   { enabled: true, useNetlink: true }
        ethtool:    { enabled: true }
        buddyinfo:  { enabled: true }
        mountstats: { enabled: true }
        ksmd:       { enabled: true }
        processes:  { enabled: true }
        systemd:    { enabled: true, units: [kubelet.service, crio.service] }
      ignoredNetworkDevices: ["veth.*", "[a-f0-9]{15}", "ovn-k8s-mp[0-9]*"]
```

## Bundled Dashboard Catalogue

CMO ships 12 bundled dashboards in two multi-document YAML files. These serve as
reference implementations for creating new dashboards.

### File: `0000_90_cluster-monitoring-operator_01-dashboards.yaml` (3 dashboards)

#### 1. Node Exporter / USE Method / Cluster

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-node-cluster-rsrc-use` |
| UID | `3e97d1d02672cdd0861f4c97c64f89b2` |
| Template vars | `$datasource` |
| Panels | 9 graphs |
| Metric source | Recording rules over node-exporter |

Applies the [USE Method](http://www.brendangregg.com/usemethod.html) (Utilisation,
Saturation, Errors) across all nodes simultaneously. Each panel divides per-instance
values by the cluster total to show proportional contribution.

| Panel | Recording rule / Expression | Y-axis |
|-------|---------------------------|--------|
| CPU Utilisation | `instance:node_cpu_utilisation:rate1m * instance:node_num_cpu:sum` | percentunit |
| CPU Saturation (Load1) | `instance:node_load1_per_cpu:ratio` | short |
| Memory Utilisation | `instance:node_memory_utilisation:ratio` | percentunit |
| Memory Saturation | `instance:node_vmstat_pgmajfault:rate1m` | ops |
| Network Utilisation (rx/tx) | `instance:node_network_{receive,transmit}_bytes_excluding_lo:rate1m` | Bps |
| Network Saturation (drops) | `instance:node_network_{receive,transmit}_drop_excluding_lo:rate1m` | ops |
| Disk IO Utilisation | `instance_device:node_disk_io_time_seconds:rate1m` | percentunit |
| Disk IO Saturation | `instance_device:node_disk_io_time_weighted_seconds:rate1m` | short |
| Disk Space Utilisation | `1 - node_filesystem_avail_bytes / node_filesystem_size_bytes` | percentunit |

#### 2. Node Exporter / USE Method / Node

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-node-rsrc-use` |
| UID | `fac67cfbe174d3ef53eb473d73d9212f` |
| Template vars | `$datasource`, `$role`, `$instance` |
| Panels | 9 graphs (same as Cluster, filtered to one node) |
| Metric source | Same recording rules, filtered by `instance="$instance"` |

Drill-down companion to the cluster-wide USE dashboard. The `$role` variable
filters nodes by `kube_node_role` label, and `$instance` selects a single node.

#### 3. Prometheus / Overview

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-prometheus` |
| UID | (empty) |
| Template vars | `$datasource`, `$job`, `$instance` |
| Panels | 1 table + 9 graphs |
| Metric source | Prometheus self-monitoring metrics |

Monitors the Prometheus instances themselves. Key panels:

| Panel | Expression | What it shows |
|-------|-----------|---------------|
| Prometheus Stats | `prometheus_build_info`, `process_start_time_seconds` | Version, uptime |
| Target Sync | `rate(prometheus_target_sync_length_seconds_sum[5m]) * 1e3` | Target discovery latency (ms) |
| Targets | `prometheus_sd_discovered_targets` | Discovered target count |
| Scrape Interval | `rate(interval_length_seconds_sum[5m]) / rate(count[5m])` | Actual vs configured interval |
| Scrape failures | 5 targets: exceeded body/sample limit, duplicate timestamp, out of bounds/order | Scrape error rates |
| Appended Samples | `rate(prometheus_tsdb_head_samples_appended_total[5m])` | Ingestion rate |
| Head Series | `prometheus_tsdb_head_series` | Active time series count |
| Head Chunks | `prometheus_tsdb_head_chunks` | In-memory chunk count |
| Query Rate | `rate(prometheus_engine_query_duration_seconds_count{slice="inner_eval"}[5m])` | Queries/s |
| Stage Duration | `max(prometheus_engine_query_duration_seconds{quantile="0.9"}) * 1e3` | p90 query latency (ms) |

### File: `0000_90_cluster-monitoring-operator_02-dashboards.yaml` (9 dashboards)

#### 4. Kubernetes / Networking / Cluster

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-cluster-total` |
| UID | `ff635a025bcfea7bc3dd4f508990a3e9` |
| Template vars | `$resolution`, `$interval`, `$datasource` |
| Panels | 5 rows, 12 graphs, 1 table |
| Metric source | cadvisor `container_network_*`, node-exporter `node_netstat_Tcp*` |

Cluster-wide network overview. Uses `irate()` with `[$interval:$resolution]`
for adjustable granularity. TCP retransmit panels use `node_netstat_Tcp_RetransSegs`
and `node_netstat_TcpExt_TCPSynRetrans`. All traffic grouped `by (namespace)`.

#### 5. Kubernetes / Networking / Namespace (Pods)

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-namespace-by-pod` |
| UID | `8b7a8b326d7a6f1f04244066368c67af` |
| Template vars | `$datasource`, `$namespace`, `$resolution`, `$interval` |
| Panels | 4 rows, 2 gauges, 6 graphs, 1 table |
| Metric source | cadvisor `container_network_*` |

Per-namespace network drill-down. Gauge panels show current rx/tx byte rates.
All traffic grouped `by (pod)` within the selected namespace.

#### 6. Kubernetes / Networking / Pod

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-pod-total` |
| UID | `7a18067ce943a40ae25454675c19ff5c` |
| Template vars | `$datasource`, `$namespace`, `$pod`, `$resolution`, `$interval` |
| Panels | 4 rows, 2 gauges, 6 graphs |
| Metric source | cadvisor `container_network_*` |

Single-pod network detail. Same panels as namespace but filtered to one pod.

#### 7. Kubernetes / Compute Resources / Cluster

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-k8s-resources-cluster` |
| UID | `efa86fd1d0c121a26444b636a3f509a8` |
| Template vars | `$datasource` |
| Panels | 6 singlestats, 16 graphs/tables |
| Metric source | Recording rules, cadvisor, kube-state-metrics |

Cluster-wide resource overview with six top-level singlestats:
- CPU Utilisation (`cluster:node_cpu:ratio_rate5m`)
- CPU Requests/Limits Commitment (requests sum / allocatable sum)
- Memory Utilisation, Requests/Limits Commitment (same pattern)

Tables show per-namespace CPU and memory quota (usage, requests, limits, ratios).
Network and storage IO panels round out the view.

#### 8. Kubernetes / Compute Resources / Namespace (Pods)

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-k8s-resources-namespace` |
| UID | `85a562078cdf77779eaa1add43ccec1e` |
| Template vars | `$datasource`, `$namespace` |
| Panels | 4 singlestats, 14 graphs/tables |
| Metric source | Recording rules, cadvisor, kube-state-metrics |

Namespace-level resource drill-down. Singlestats show CPU/memory utilisation
from requests and limits. Tables list per-pod resource usage and quota.

#### 9. Kubernetes / Compute Resources / Node (Pods)

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-k8s-resources-node` |
| UID | `200ac8fdbfbb74b39aff88118e4d1c2c` |
| Template vars | `$datasource`, `$role`, `$node` |
| Panels | 2 graphs, 2 tables |
| Metric source | Recording rules, kube-state-metrics |

Shows CPU and memory usage for all pods on a single node.
Uses `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate`
filtered by node. Tables show requests/limits per pod.

#### 10. Kubernetes / Compute Resources / Pod

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-k8s-resources-pod` |
| UID | `6581e46e4e5c7ba40a07646395ef7b23` |
| Template vars | `$datasource`, `$namespace`, `$pod` |
| Panels | 13 graphs/tables |
| Metric source | Recording rules, cadvisor, kube-state-metrics |

Single-pod resource detail. Unique panel: **CPU Throttling** using
`container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total`.
Also includes per-container IOPS and throughput panels.

#### 11. Kubernetes / Compute Resources / Workload

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-k8s-resources-workload` |
| UID | `a164a7f0339f99e89cea5cb47e9be617` |
| Template vars | `$datasource`, `$namespace`, `$type`, `$workload` |
| Panels | 13 graphs/tables |
| Metric source | Recording rules, cadvisor, kube-state-metrics |

Workload-level view (Deployment, StatefulSet, DaemonSet, Job). Uses
`namespace_workload_pod:kube_pod_owner:relabel` to group pods by workload.
Network panels show per-pod bandwidth within the workload.

#### 12. Kubernetes / Compute Resources / Namespace (Workloads)

| Field | Value |
|-------|-------|
| ConfigMap | `dashboard-k8s-resources-workloads-namespace` |
| UID | `a87fb0d919ec0ea5f6543124e16c42a5` |
| Template vars | `$datasource`, `$namespace`, `$type` |
| Panels | 13 graphs/tables |
| Metric source | Recording rules, cadvisor, kube-state-metrics |

Namespace-level view grouped by workload. Similar to dashboard 8 but aggregated
by workload rather than pod. Includes `kube_resourcequota` for quota comparison.

### Dashboard Topic Selection Guide

| Topic area | Existing dashboard(s) | Metric sources | Key patterns |
|-----------|----------------------|----------------|--------------|
| **Node USE Method** | 1, 2 | node-exporter recording rules | Single recording rule per panel, ratio |
| **Network by namespace** | 4, 5 | cadvisor `container_network_*` | `irate()` with `$interval:$resolution` |
| **Network by pod** | 6 | cadvisor `container_network_*` | Same, filtered to single pod |
| **Compute resources cluster** | 7 | Recording rules + kube-state-metrics | Singlestats + tables |
| **Compute resources namespace** | 8 | Recording rules + cadvisor + KSM | Drill-down by namespace |
| **Compute resources node** | 9 | Recording rules + KSM | Drill-down by node |
| **Compute resources pod** | 10 | Recording rules + cadvisor + KSM | CPU throttling, per-container |
| **Compute resources workload** | 11, 12 | Recording rules + cadvisor + KSM | `namespace_workload_pod:kube_pod_owner:relabel` join |
| **Prometheus self-monitoring** | 3 | Prometheus `prometheus_*` | Self-referential metrics |
| **Kernel stack / AF_PACKET** | standalone | node-exporter (incl. disabled collectors) | `rate()`, `topk()`, custom sidecar |

When creating a new dashboard, identify which row in this table your topic falls
under, then use the matching existing dashboard as a starting template.

## Job Filter Strategy

The AF_PACKET dashboard uses three different job filter patterns depending on
whether the metric comes from the CMO-managed node-exporter, a custom sidecar,
or either. This strategy ensures the dashboard works with any combination of
exporter deployments.

| Pattern | When to use | Example panels |
|---------|-------------|----------------|
| `job="node-exporter"` | Metrics from default-enabled collectors only (cpu, netdev, conntrack, netstat, sockstat, meminfo) | 101, 102, 126, 106-114, 116, 118 |
| `job=~"node-exporter.*"` | Metrics that may come from either the CMO exporter or a custom sidecar with a similar job name (e.g., `node-exporter-sidecar`) | 132, 104, 130 (softirqs), 120, 121 (pressure) |
| No job filter | Metrics from disabled-by-default collectors that will only exist on a custom sidecar (which may have any job name) | 134, 135 (interrupts), 125 (zoneinfo) |

**Rationale**: When a custom sidecar node-exporter is deployed alongside the
CMO-managed one (to expose `interrupts`, `softirqs`, `zoneinfo`), it typically
gets a different job label (e.g., `node-exporter-sidecar`). Using `job=~"node-exporter.*"`
matches both. For metrics that can only come from the sidecar (interrupts, zoneinfo),
omitting the job filter avoids needing to know the sidecar's exact job name.

## AF_PACKET Dashboard Reference

Source: `dashboard-kernel-stack-af-packet-standalone.yaml`
(from [midu16/cluster-monitoring-operator](https://github.com/midu16/cluster-monitoring-operator))

**Title**: "Node Exporter / Kernel Stack Cost & AF_PACKET"
**UID**: `node-kernel-stack-af-packet`
**Purpose**: A single pane of glass to diagnose every source of kernel overhead
on CPUs isolated for high-performance packet processing (DPDK, AF_PACKET, XDP).
It contains **33 panels** across **10 sections**, including explanatory text
panels with markdown content.

### The Problem It Solves

In DPDK/AF_PACKET deployments, specific CPUs are **isolated** from the Linux
scheduler and **pinned** to userspace applications that poll NICs directly.
The contract is: these CPUs spend 100% of their time in userspace poll loops.
Any kernel work -- even microseconds -- causes **packet loss**, **jitter**,
and **throughput degradation**.

| Root cause | Symptom | Dashboard section |
|------------|---------|-------------------|
| IRQ affinity misconfiguration | NIC queue IRQ lands on isolated CPU | Section 2 (Panels 134/135) |
| AF_PACKET fanout misconfiguration | Kernel processes packets on wrong CPU | Section 3 (Panels 132/104/130) |
| NAPI polling on wrong CPU | NET_RX softirq runs on isolated CPU | Section 1 (Panel 102) + Section 3 |
| Conntrack table exhaustion | Silent packet drops when table is full | Section 5 (Panels 109/110) |
| TCP/UDP buffer overflow | Application socket buffer full, kernel drops | Section 6 (Panels 112/113/114) |
| NUMA misplacement | Hugepages allocated on remote NUMA node | Section 10 (Panel 125) |
| Memory pressure | Direct reclaim stalls all tasks | Section 9 (Panels 120/121) |
| Hugepage exhaustion | DPDK cannot allocate mbuf pools | Section 8 (Panel 118) |

### Investigation Workflow (Top-to-Bottom)

**Step 1 -- Is the kernel stealing CPU cycles?** (Section 1, Panels 101/102/126)

Three side-by-side panels. For each isolated CPU:
- `system%` > 5% → kernel syscalls or page faults consuming time
- `softirq%` > 1% → kernel networking (NET_RX/NET_TX) processing packets
- `irq%` > 0.5% → hardware interrupts firing (NIC queues, timers)

If all three are near 0, the kernel is not the problem. Stop here.

**Step 2 -- Which device is causing interrupts?** (Section 2, Panels 134/135)

Shows exactly which NIC queue IRQ lands on which CPU. The `devices` label reveals
names like `eno1np0-TxRx-3`. If this queue hits an isolated CPU, fix IRQ affinity:
`echo <housekeeping_cpumask> > /proc/irq/<N>/smp_affinity`.

**Step 3 -- Which kernel function consumes cycles?** (Section 3, Panels 132/104/130)

Stacked softirq-type chart per CPU:
- `NET_RX` dominating → NAPI poll running (kernel processing received packets)
- `NET_TX` dominating → kernel transmitting (XDP_REDIRECT, AF_PACKET TX)
- `TIMER` dominating → check `nohz_full` kernel parameter
- `RCU` dominating → check `rcu_nocbs` kernel parameter

**Step 4 -- Is the NIC saturated?** (Section 4, Panels 106/107)

Throughput approaching line rate with rising `receive_drop` or `receive_errs` =
NIC-level saturation (hardware/driver issue, not kernel CPU issue).

**Step 5 -- Is conntrack dropping?** (Section 5, Panels 109/110)

`entries` approaching `limit` with `drop > 0` = connection tracking table full.
Invisible to NIC counters and application logs.

**Step 6 -- Are TCP/UDP buffers overflowing?** (Section 6, Panels 112/113/114)

Application-level packet loss: `TCPRcvQDrop` (app not reading TCP fast enough),
`ListenOverflows` (accept queue full), `Udp_RcvbufErrors` (UDP buffer full).

**Step 7 -- Is socket memory under pressure?** (Section 7, Panel 116)

Rising TCP/UDP memory with drops in Step 6 confirms buffer exhaustion.

**Step 8 -- Are hugepages available?** (Section 8, Panel 118)

`Free` = 0 means DPDK cannot allocate new memory pools. Monitor during DPDK startup.

**Step 9 -- Is the system under pressure?** (Section 9, Panels 120/121)

PSI `stalled > 0` = entire system blocked. Even 0.1% stalled time can cause
thousands of dropped packets in a 10Mpps workload.

**Step 10 -- Is memory on the wrong NUMA node?** (Section 10, Panel 125)

NUMA miss ratio > 0 = packet buffers allocated from remote node, adding ~100ns
per access.

### Complete Panel Inventory (33 panels)

```
[text  ] id= 99  y=  0  (intro markdown: what the dashboard shows)
Section 1: Isolated / pinned CPUs — kernel work % and breakdown
[row   ] id=100  y=  3  "Isolated / pinned CPUs (e.g. DPDK) — kernel work % and breakdown by function type"
[graph ] id=101  y=  4  System time % per CPU (kernel / syscalls)          w=8
[graph ] id=102  y=  4  Softirq time % per CPU (e.g. NET_RX / NET_TX)     w=8
[graph ] id=126  y=  4  IRQ time % per CPU (hard interrupts)               w=8

Section 2: IRQ per CPU — map interrupts to isolate DPDK CPUs
[row   ] id=136  y= 12  "IRQ per CPU (which device hits which CPU) — map interrupts to isolate DPDK CPUs"
[text  ] id=133  y= 13  (explains node_intr_total / interrupts collector)
[graph ] id=134  y= 14  IRQ rate per CPU by device (all)                   w=24
[graph ] id=135  y= 22  IRQ rate per CPU (network devices only — NIC queues) w=24 stacked

Section 3: Softirq breakdown — which kernel function consumes cycles
[row   ] id=103  y= 30  "Softirq breakdown by function type (NET_RX, NET_TX, etc.)"
[text  ] id=131  y= 31  (explains invocation counts vs CPU time)
[graph ] id=132  y= 33  Per CPU: softirq type as % of invocations          w=24 stacked
[graph ] id=104  y= 41  NET_RX softirq rate per CPU                        w=12
[graph ] id=130  y= 41  All softirq types per CPU (invocation rate)        w=12 stacked

Section 4: Network throughput and drops
[row   ] id=105  y= 49
[graph ] id=106  y= 50  Network throughput (bytes/s)                       w=12
[graph ] id=107  y= 50  Receive drops and errors (rate)                    w=12

Section 5: Conntrack — table full → packet drops
[row   ] id=108  y= 58  "Conntrack (nf_conntrack) — table full → packet drops"
[graph ] id=109  y= 59  Conntrack entries vs limit                         w=12
[graph ] id=110  y= 59  Conntrack drops (increase 1m)                      w=12

Section 6: Netstat — TCP/UDP buffer and listen-queue drops
[row   ] id=111  y= 67
[graph ] id=112  y= 68  TCP receive queue drops                            w=12
[graph ] id=113  y= 68  Listen queue overflows and drops                   w=12
[graph ] id=114  y= 76  UDP buffer errors                                  w=12

Section 7: Sockstat — socket memory
[row   ] id=115  y= 78
[graph ] id=116  y= 79  TCP/UDP socket memory (bytes)                      w=12

Section 8: Hugepages (runbook / DPDK context)
[row   ] id=117  y= 87
[graph ] id=118  y= 88  HugePages Total / Free / Rsvd                     w=12

Section 9: Pressure (memory / I/O stall)
[row   ] id=119  y= 96
[graph ] id=120  y= 97  Memory pressure                                    w=12
[graph ] id=121  y= 97  I/O and CPU pressure                               w=12

Section 10: NUMA (optional — zoneinfo / pcidevice)
[row   ] id=124  y=112
[graph ] id=125  y=113  NUMA miss ratio (per node)                         w=12
```

### Linux Kernel Packet Path → Dashboard Section Mapping

```
NIC hardware                     Section 2: IRQ per CPU
  │                              (which device hits which CPU)
  ▼
IRQ top-half (ISR)               Section 1: IRQ time % per CPU
  │ schedules softirq             (mode="irq" fraction)
  ▼
NET_RX softirq (NAPI poll)       Section 1: Softirq time % per CPU
  │ processes packet batch         Section 3: Softirq breakdown by type
  │ (up to netdev_budget=300)      (NET_RX vs NET_TX vs TIMER vs ...)
  ▼
netfilter / conntrack             Section 5: Conntrack entries vs limit
  │ tracks connection state        (table full → drops)
  ▼
TCP/UDP socket buffer             Section 6: Netstat drops
  │ queues for application         (TCPRcvQDrop, ListenOverflows, UdpRcvbufErrors)
  ▼
Application read()/recv()         Section 7: Socket memory
                                   (TCP/UDP mem_bytes)
Device throughput & drops         Section 4: Network throughput

Parallel concerns:
  Memory                          Section 8: HugePages + Section 9: PSI pressure
  NUMA topology                   Section 10: NUMA miss ratio
```

### Complete PromQL Expression Reference

| Panel ID | Title | PromQL | Job filter | Collector | Y-axis |
|----------|-------|--------|------------|-----------|--------|
| 101 | System time % per CPU | `sum(rate(node_cpu_seconds_total{job="node-exporter", mode="system"}[1m])) by (instance,cpu) / sum(rate(node_cpu_seconds_total{job="node-exporter"}[1m])) by (instance,cpu)` | `job="node-exporter"` | cpu (default) | percentunit |
| 102 | Softirq time % per CPU | `sum(rate(node_cpu_seconds_total{job="node-exporter", mode="softirq"}[1m])) by (instance,cpu) / sum(rate(node_cpu_seconds_total{job="node-exporter"}[1m])) by (instance,cpu)` | `job="node-exporter"` | cpu (default) | percentunit |
| 126 | IRQ time % per CPU | `sum(rate(node_cpu_seconds_total{job="node-exporter", mode="irq"}[1m])) by (instance,cpu) / sum(rate(node_cpu_seconds_total{job="node-exporter"}[1m])) by (instance,cpu)` | `job="node-exporter"` | cpu (default) | percentunit |
| 134 | IRQ rate per CPU (all) | `topk(100, rate(node_intr_total{}[1m]))` | none | stat / interrupts | ops |
| 135 | IRQ rate per CPU (NIC) | `topk(100, rate(node_interrupts_total{devices=~".*TxRx.*\|.*eno.*\|.*net.*"}[1m]))` | none | interrupts (**disabled**) | ops |
| 132 | Softirq type % per CPU | `sum(rate(node_softirqs_functions_total{job=~"node-exporter.*"}[1m])) by (instance,cpu,type) / sum(rate(node_softirqs_functions_total{job=~"node-exporter.*"}[1m])) by (instance,cpu)` | `job=~"node-exporter.*"` | softirqs (**disabled**) | percentunit |
| 104 | NET_RX rate per CPU | `rate(node_softirqs_functions_total{job=~"node-exporter.*", type="NET_RX"}[1m])` | `job=~"node-exporter.*"` | softirqs (**disabled**) | ops |
| 130 | All softirq types per CPU | `rate(node_softirqs_functions_total{job=~"node-exporter.*"}[1m])` | `job=~"node-exporter.*"` | softirqs (**disabled**) | ops |
| 106 | Network throughput | `rate(node_network_receive_bytes_total{job="node-exporter", device!="lo"}[1m])` + tx | `job="node-exporter"` | netdev (default) | Bps |
| 107 | Receive drops/errors | `rate(node_network_receive_drop_total{job="node-exporter", device!="lo"}[1m])` + errs | `job="node-exporter"` | netdev (default) | ops |
| 109 | Conntrack entries vs limit | `node_nf_conntrack_entries{job="node-exporter"}` + `node_nf_conntrack_entries_limit{job="node-exporter"}` | `job="node-exporter"` | conntrack (default) | short |
| 110 | Conntrack drops | `increase(node_nf_conntrack_stat_drop{job="node-exporter"}[1m])` + early_drop | `job="node-exporter"` | conntrack (default) | short |
| 112 | TCP rcvq drops | `increase(node_netstat_TcpExt_TCPRcvQDrop{job="node-exporter"}[1m])` | `job="node-exporter"` | netstat (default) | short |
| 113 | Listen queue drops | `increase(node_netstat_TcpExt_ListenOverflows{job="node-exporter"}[1m])` + ListenDrops | `job="node-exporter"` | netstat (default) | short |
| 114 | UDP buffer errors | `increase(node_netstat_Udp_RcvbufErrors{job="node-exporter"}[1m])` + SndbufErrors | `job="node-exporter"` | netstat (default) | short |
| 116 | TCP/UDP socket memory | `node_sockstat_TCP_mem_bytes{job="node-exporter"}` + UDP | `job="node-exporter"` | sockstat (default) | bytes |
| 118 | HugePages Total/Free/Rsvd | `node_memory_HugePages_Total{job="node-exporter"}` + Free + Rsvd | `job="node-exporter"` | meminfo (default) | short |
| 120 | Memory pressure | `rate(node_pressure_memory_stalled_seconds_total{job=~"node-exporter.*"}[1m])` + waiting | `job=~"node-exporter.*"` | pressure (default) | percentunit |
| 121 | I/O + CPU pressure | `rate(node_pressure_io_stalled_seconds_total{job=~"node-exporter.*"}[1m])` + cpu_waiting | `job=~"node-exporter.*"` | pressure (default) | percentunit |
| 125 | NUMA miss ratio | `sum(rate(node_zoneinfo_numa_miss_total{}[1m])) by (instance,node) / (hit + miss)` | none | zoneinfo (**disabled**) | percentunit |

### Text Panel Content

The dashboard includes explanatory markdown text panels that help operators understand
what each section shows. These should be preserved or adapted in new dashboards:

| Panel ID | Section | Content summary |
|----------|---------|----------------|
| 99 | Header | Explains the dashboard purpose: for isolated/pinned CPUs (DPDK), shows kernel work %, broken down by function type |
| 133 | Section 2 | Explains `node_intr_total` from `/proc/interrupts`, notes the `interrupts` collector is disabled by default due to cardinality, and how to use the panels to tune IRQ affinity |
| 131 | Section 3 | Explains that softirq counters show invocation counts (not CPU time), so the stacked chart shows share of invocations; combine with softirq time % to confirm kernel networking is the culprit |

### Severity Classification

| Metric / Panel | OK | Warning | Critical | Remediation |
|----------------|----|---------| ---------|-------------|
| System time % (101) | < 1% on isolated CPUs | 1-5% | > 5% | Identify source via softirq/IRQ panels |
| Softirq time % (102) | < 0.5% on isolated CPUs | 0.5-2% | > 2% | Check IRQ affinity, `nohz_full`, `rcu_nocbs` |
| IRQ time % (126) | < 0.1% on isolated CPUs | 0.1-0.5% | > 0.5% | Fix `smp_affinity`, check MSI-X queue count |
| IRQ rate on isolated CPU (134/135) | 0 ops/s | Any non-zero | Sustained > 1000/s | `/proc/irq/<N>/smp_affinity_list` |
| NET_RX % of softirqs (132) | < 10% on isolated CPUs | 10-50% | > 50% | IRQ affinity or AF_PACKET fanout |
| Network rx drops (107) | 0 | Any sustained > 0 | > 100/s | Ring buffer size, NIC offloads, RPS |
| Conntrack entries (109) | < 80% of limit | 80-95% | > 95% or drops > 0 | Increase `nf_conntrack_max` |
| TCPRcvQDrop (112) | 0 | Any > 0 | Sustained > 0 | Increase `rmem_max`, fix application |
| ListenOverflows (113) | 0 | Any > 0 | Sustained > 0 | Increase `somaxconn`, fix `accept()` loop |
| Udp RcvbufErrors (114) | 0 | Any > 0 | Sustained > 0 | Increase `SO_RCVBUF`, `rmem_max` |
| HugePages Free (118) | > 10% of Total | 5-10% | < 5% or 0 | Allocate more hugepages |
| Memory stalled (120) | 0 | Any > 0 | > 1% | Add RAM, reduce memory pressure |
| NUMA miss ratio (125) | 0 | > 0 and < 5% | > 5% | Fix `numactl`, DPDK `--socket-mem` |

### Metrics-to-Collectors Map

| Dashboard Section | Metrics | Collector | Default? | Job filter |
|-------------------|---------|-----------|----------|------------|
| System/softirq/IRQ time % | `node_cpu_seconds_total` | cpu | Yes | `job="node-exporter"` |
| IRQ per CPU (all) | `node_intr_total` | stat / interrupts | Partial | none |
| IRQ per CPU (NIC) | `node_interrupts_total` | interrupts | **No** | none |
| Softirq breakdown | `node_softirqs_functions_total` | softirqs | **No** | `job=~"node-exporter.*"` |
| Network throughput | `node_network_{receive,transmit}_bytes_total` | netdev | Yes | `job="node-exporter"` |
| Network drops/errors | `node_network_receive_{drop,errs}_total` | netdev | Yes | `job="node-exporter"` |
| Conntrack entries/limit | `node_nf_conntrack_{entries,entries_limit}` | conntrack | Yes | `job="node-exporter"` |
| Conntrack drops | `node_nf_conntrack_stat_{drop,early_drop}` | conntrack | Yes | `job="node-exporter"` |
| TCP rcvq drops | `node_netstat_TcpExt_TCPRcvQDrop` | netstat | Yes | `job="node-exporter"` |
| Listen queue drops | `node_netstat_TcpExt_Listen{Overflows,Drops}` | netstat | Yes | `job="node-exporter"` |
| UDP buffer errors | `node_netstat_Udp_{RcvbufErrors,SndbufErrors}` | netstat | Yes | `job="node-exporter"` |
| Socket memory | `node_sockstat_{TCP,UDP}_mem_bytes` | sockstat | Yes | `job="node-exporter"` |
| HugePages | `node_memory_HugePages_{Total,Free,Rsvd}` | meminfo | Yes | `job="node-exporter"` |
| Memory pressure | `node_pressure_memory_{stalled,waiting}_seconds_total` | pressure | Yes | `job=~"node-exporter.*"` |
| I/O + CPU pressure | `node_pressure_{io_stalled,cpu_waiting}_seconds_total` | pressure | Yes | `job=~"node-exporter.*"` |
| NUMA miss ratio | `node_zoneinfo_numa_{miss,hit}_total` | zoneinfo | **No** | none |

### Deployment Prerequisites for Disabled Collectors

| Collector | Required by panels | CMO-configurable? | How to enable |
|-----------|-------------------|-------------------|---------------|
| `interrupts` | 134, 135 | No | Custom sidecar with `--collector.interrupts` |
| `softirqs` | 132, 104, 130 | No | Custom sidecar with `--collector.softirqs` |
| `zoneinfo` | 125 | No | Custom sidecar with `--collector.zoneinfo` |

## Common Issues

1. **"No data" on network panels**: Default CMO excludes all netdev/netclass devices (`^.*$`). Re-enable via `cluster-monitoring-config`.
2. **"No data" on softirq/interrupt panels**: Collectors disabled by default, not CMO-configurable. Deploy custom sidecar.
3. **Overlapping panels**: Ensure `gridPos.y` values don't overlap. Each panel's `y` must be >= previous panel's `y + h`.
4. **Panel ID collisions**: Every panel needs a unique `"id"`. Grafana silently breaks on duplicates.
5. **Missing annotations**: Without `capability.openshift.io/name: Console` and `console.openshift.io/dashboard: "true"`, the OCP Console won't discover the dashboard.
6. **HugePages in wrong unit**: `node_memory_HugePages_*` are page counts (not bytes). The page size is in `node_memory_Hugepagesize_bytes`.

## Diagnostic Examples

### Example 1: Diagnosing Kernel Overhead on an Isolated CPU

```
Panel 101 shows cpu14 at 8% system time → check Panel 102
Panel 102 shows cpu14 at 6% softirq → check Panel 132
Panel 132 shows cpu14 has 85% NET_RX → NAPI poll is running
Panel 135 shows eno1np0-TxRx-6 hitting cpu14 at 45k IRQs/s

Diagnosis: NIC queue 6 has IRQ affinity set to cpu14 (isolated for DPDK).
Fix: echo <housekeeping_mask> > /proc/irq/$(grep eno1np0-TxRx-6 /proc/interrupts | cut -d: -f1)/smp_affinity_list
```

### Example 2: Silent Conntrack Packet Loss

```
Panel 109 shows entries at 260,000 vs limit 262,144 (99.2%)
Panel 110 shows drop increasing by ~50/min and early_drop by ~200/min

Diagnosis: Conntrack table nearly full. New connections dropped silently.
Fix: sysctl -w net.netfilter.nf_conntrack_max=524288
     OR: bypass conntrack for DPDK traffic using iptables -t raw -A PREROUTING -i <iface> -j NOTRACK
```

### Example 3: Creating a Simple Network Throughput Panel

```json
{"datasource": "$datasource", "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}, "id": 1,
 "legend": {"show": true, "alignAsTable": true, "current": true, "values": true},
 "lines": true,
 "targets": [
   {"expr": "rate(node_network_receive_bytes_total{job=\"node-exporter\", device!=\"lo\"}[1m])",
    "format": "time_series", "legendFormat": "{{instance}} {{device}} rx", "refId": "A"},
   {"expr": "rate(node_network_transmit_bytes_total{job=\"node-exporter\", device!=\"lo\"}[1m])",
    "format": "time_series", "legendFormat": "{{instance}} {{device}} tx", "refId": "B"}],
 "title": "Network throughput (bytes/s)", "type": "graph",
 "yaxes": [{"format": "Bps", "min": 0, "show": true}, {"show": false}]}
```

On a 25Gbps NIC, max throughput ~3.1 GB/s. If throughput approaches line rate
and `receive_drop` rises, the NIC is saturated.

### Example 4: Custom Sidecar Node-Exporter for Disabled Collectors

```yaml
image: quay.io/prometheus/node-exporter:v1.9.1
args:
  - --web.listen-address=0.0.0.0:9101
  - --path.procfs=/host/proc
  - --collector.disable-defaults
  - --collector.zoneinfo
  - --collector.interrupts
  - --collector.softirqs
```

Deploy as a sidecar DaemonSet with its own ServiceMonitor. Enables all three
disabled collectors needed by the AF_PACKET dashboard without affecting the
CMO-managed instance. Use `job=~"node-exporter.*"` in PromQL to match both
the standard and sidecar exporters.

## Tips for Effective Dashboard Creation

1. Start from the AF_PACKET dashboard as a template for standalone dashboards
2. Always include the `$datasource` templating variable
3. Use `row` panels with descriptive titles to group related metrics — include context like "which device hits which CPU" not just "IRQ"
4. Add `text` panels (markdown) before complex sections explaining what the graphs show and why — this makes the dashboard self-documenting
5. Use `topk(N, ...)` for high-cardinality metrics (interrupts, ethtool)
6. Prefer `rate()` over `increase()` for counters — except for event-counting panels
7. Always filter `device!="lo"` for network metrics
8. Check the minimal collection profile before using any metric
9. Use the correct job filter strategy (see Job Filter Strategy section):
   - `job="node-exporter"` for default-enabled collectors
   - `job=~"node-exporter.*"` for metrics that may come from either the CMO exporter or a sidecar
   - No job filter for metrics that only exist on a custom sidecar
10. When using disabled-by-default collectors, include a text panel explaining the dependency and how to enable them

## See Also

- [midu16/cluster-monitoring-operator](https://github.com/midu16/cluster-monitoring-operator) — source repo with AF_PACKET standalone dashboard
- `manifests/dashboard-kernel-stack-af-packet-standalone.yaml` — the reference standalone dashboard (33 panels, 10 sections)
- `manifests/0000_90_cluster-monitoring-operator_01-dashboards.yaml` — bundled upstream dashboards (USE Method, Prometheus)
- `manifests/0000_90_cluster-monitoring-operator_02-dashboards.yaml` — bundled upstream dashboards (Networking, Compute Resources)
- `assets/node-exporter/daemonset.yaml` — CMO node-exporter DaemonSet definition
- `assets/node-exporter/minimal-service-monitor.yaml` — minimal collection profile allowlist (v1.10.2)
- `pkg/manifests/manifests.go` — `updateNodeExporterArgs()` function
- [prometheus/node_exporter](https://github.com/prometheus/node_exporter) — collector source code
