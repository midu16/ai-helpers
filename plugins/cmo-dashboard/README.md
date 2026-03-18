# cmo-dashboard

Create and analyze Grafana dashboard ConfigMaps for the OpenShift cluster-monitoring-operator (CMO).

Covers all metric sources in the CMO stack: node-exporter, cadvisor, kube-state-metrics,
Prometheus self-monitoring, and recording rules. Understands all 12 bundled dashboards
shipped with CMO and can create new standalone dashboards for any monitoring topic.

## Commands

### `create-dashboard`

Generate a standalone Grafana dashboard ConfigMap YAML from a natural-language description.
Selects appropriate metrics from any CMO source, PromQL expressions, panel types, template
variables, and layout, then produces a valid YAML file ready for inclusion in CMO manifests.

```
/cmo-dashboard:create-dashboard <topic> [--panels <descriptions>] [--source <sources>] [--template-vars <vars>]
```

Supported metric sources: `node-exporter`, `cadvisor`, `kube-state-metrics`, `prometheus`, `recording-rules`.

### `analyze-dashboard`

Analyze an existing dashboard YAML for structural issues, metric source availability,
recording rule validity, collection profile compatibility, and diagnostic coverage gaps.

```
/cmo-dashboard:analyze-dashboard <path-to-dashboard-yaml> [--check-profile minimal|full] [--explain]
```

Handles both standalone and bundled multi-document YAML files.

## Skills

### grafana-dashboard-reference

Comprehensive reference material consumed by both commands:

- **Dashboard YAML skeleton** and naming conventions
- **Panel templates**: row, text, graph, singlestat, table (with examples from bundled dashboards)
- **Template variable patterns**: datasource, namespace, pod, node, role, workload, resolution
- **PromQL pattern catalogue**: rate/irate, topk, stacked ratio, instant table, commitment ratio, and more
- **Metric sources**: node-exporter collectors, cadvisor container metrics, kube-state-metrics, Prometheus self-monitoring
- **Recording rules**: all ~30 rules directly consumed by bundled dashboards with raw expressions
- **Bundled dashboard catalogue**: all 12 dashboards with panel lists, metric sources, and template variables
  - Node Exporter / USE Method (Cluster + Node)
  - Prometheus / Overview
  - Kubernetes / Networking (Cluster, Namespace, Pod)
  - Kubernetes / Compute Resources (Cluster, Namespace, Node, Pod, Workload, Namespace-Workloads)
- **Node-exporter collector reference** (v1.9.1): default-enabled, disabled-by-default, CMO-configurable
- **CMO architecture**: delivery pipeline, collection profiles (full + minimal with exact allowlist), CMO-configurable collectors
- **AF_PACKET dashboard**: kernel packet path mapping, severity classification, investigation workflow

## Prerequisites

- Familiarity with the cluster-monitoring-operator repository
- Understanding of Kubernetes ConfigMaps and YAML
- Basic PromQL knowledge
