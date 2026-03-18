# telco-reference

Create, inspect, and compare OpenShift telco reference configurations across releases.

Covers three use models from the [openshift-kni/telco-reference](https://github.com/openshift-kni/telco-reference) repository:

- **telco-core** — Large multi-node clusters for vDU/vCU workloads (since 4.14)
- **telco-hub** — Management clusters running ACM, GitOps, TALM (since 4.18)
- **telco-ran** — Edge SNO/3-node DU sites managed via ZTP (since 4.19)

## Commands

| Command | Description |
|---------|-------------|
| `create-config` | Generate a telco reference configuration for a specific use model and release |
| `inspect` | Inspect and explain telco reference CRs with dependency and compatibility analysis |
| `upgrade-analysis` | Analyze migration path between two releases with risk assessment and migration plan |
| `feature-summary` | Summarize all features, operators, and capabilities available in a release branch |

## Skills

| Skill | Description |
|-------|-------------|
| `telco-reference-catalogue` | Complete reference covering all branches (4.14–4.21+main), CR catalogues, operator inventories, version differences, upgrade paths, and deployment architecture |

## Usage

```bash
# Generate a core config for 4.21 with optional cert-manager
/telco-reference:create-config core 4.21 --operators "cert-manager,logging"

# Analyze upgrade from 4.19 to 4.21
/telco-reference:upgrade-analysis 4.19 4.21 --use-model core

# Summarize what's available in 4.20
/telco-reference:feature-summary 4.20

# Inspect a CR for compatibility
/telco-reference:inspect PerformanceProfile --release 4.21
```
