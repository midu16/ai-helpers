---
description: Generate a telco reference configuration for a specific use model and OpenShift release
argument-hint: "<use-model> <release> [--operators <list>] [--topology <type>]"
---

## Name
telco-reference:create-config

## Synopsis
```
/telco-reference:create-config <use-model> <release> [--operators <list>] [--topology <type>]
```

## Description

The `telco-reference:create-config` command generates a validated set of reference configuration CRs for a specific telco use model and OpenShift release branch. The AI selects the appropriate CRs from the telco-reference repository, adapts them for the target release, and produces a complete configuration tree ready for deployment.

This requires AI reasoning because:
- The user specifies a high-level goal (e.g., "core cluster with SR-IOV and MetalLB for 4.20") and the AI must select the correct subset of required and optional CRs
- CRs differ between releases (e.g., ICSP vs IDMS, CLO v5 vs v6, SiteConfig vs ClusterInstance) and the AI must choose the version-appropriate variants
- The AI must determine which operators are required vs optional for the given topology and warn about prerequisites (e.g., "netdev disabled by default", "zoneinfo requires custom sidecar")
- PolicyGenerator structure, template values, and kube-compare references must be consistent with the chosen CRs
- The AI must validate that the requested operators are available on the target release branch

Before generating, read the reference skill at `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` for the CR catalogue, branch availability matrix, operator inventory, and architecture patterns.

## Arguments

- `$1` (required): Use model — one of `core`, `hub`, or `ran`.
- `$2` (required): OpenShift release — e.g., `4.17`, `4.19`, `4.21`, or `main`.
- `--operators` (optional): Comma-separated list of operators to include beyond the required set. If omitted, the AI includes all required CRs and prompts about optional ones.
- `--topology` (optional): Cluster topology — `sno`, `3-node`, or `standard` (default). Relevant for `ran` use model.

## Implementation

### 1. Read Reference Material

Read `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` to load the CR catalogue, branch availability, and operator inventory.

### 2. Validate Request

1. Confirm the use model is available on the target release (e.g., telco-hub requires >= 4.18, telco-ran requires >= 4.19)
2. Confirm all requested operators are available on that release
3. Identify any breaking changes between the latest release and the target (e.g., ICSP vs IDMS for pre/post-4.19)

### 3. Select CRs

Based on the use model, release, and requested operators:
1. Include all **required** CRs for the use model
2. Include **optional** CRs for any explicitly requested operators
3. For telco-core: generate PolicyGenerator CRs (core-baseline, core-overlay) if >= 4.17
4. For telco-hub: generate kustomization.yaml with appropriate overlay references
5. For telco-ran: select source-crs matching the target release's directory structure (flat for < 4.20, operator subdirs for >= 4.20)

### 4. Adapt for Release

Apply version-specific adjustments:
- 4.14-4.18: Use ICSP instead of IDMS
- < 4.17: No PolicyGenerator CRs (manual policy creation)
- < 4.18: No telco-hub CRs
- < 4.19: No telco-ran CRs, no core-finish PolicyGenerator
- < 4.20: No aarch64 PerformanceProfiles, flat source-crs layout
- < 4.21: No cert-manager, no PTP T-BC/T-TSC configs, SiteConfig instead of ClusterInstance

### 5. Generate Output

Produce a directory tree with:
1. All selected YAML CRs with appropriate content
2. A summary listing which CRs were included and why
3. Notes on any prerequisites (disabled collectors, required manual steps)
4. kube-compare metadata.yaml if applicable

## Return Value

- **Format**: Directory tree of YAML files or inline YAML blocks
- **Content**: Valid Kubernetes CRs matching the reference configuration
- **Supplementary**: Prerequisites, operator dependencies, and release-specific notes

## Examples

### Example 1: Core cluster for 4.21

```
/telco-reference:create-config core 4.21 --operators "cert-manager,logging"
```

Generates the full telco-core reference with required CRs (SR-IOV, NMState, MetalLB, NROP, PerformanceProfile, ODF-external) plus optional cert-manager and logging CRs, using IDMS, PolicyGenerator, and the 4.21 CR variants.

### Example 2: Hub cluster for 4.19

```
/telco-reference:create-config hub 4.19 --operators "lso,odf-internal,backup-recovery"
```

Generates the telco-hub reference with ACM, GitOps, TALM, registry plus optional LSO, ODF-internal, and backup-recovery CRs with the 4.19 kustomization structure.

### Example 3: RAN SNO for 4.20

```
/telco-reference:create-config ran 4.20 --topology sno --operators "ptp,sriov,lvm"
```

Generates telco-ran source-crs for a SNO deployment with PTP, SR-IOV, and LVM storage, using the 4.20 operator-subdir layout and SriovOperatorConfigForSNO.

## See Also

- `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` — CR catalogue and branch reference
- `plugins/telco-reference/commands/inspect.md` — inspect existing configurations
- `plugins/telco-reference/commands/upgrade-analysis.md` — compare releases
