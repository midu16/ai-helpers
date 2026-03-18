---
description: Inspect and explain telco reference CRs, identifying their purpose, release compatibility, and dependencies
argument-hint: "<path-or-cr-name> [--release <version>]"
---

## Name
telco-reference:inspect

## Synopsis
```
/telco-reference:inspect <path-or-cr-name> [--release <version>]
```

## Description

The `telco-reference:inspect` command analyzes one or more telco reference configuration CRs and provides a detailed explanation of their purpose, operator dependencies, release compatibility, and relationship to the broader reference configuration.

This requires AI reasoning because:
- The AI must identify which use model and operator category a CR belongs to (e.g., a SriovNetworkNodePolicy belongs to telco-core/ran SR-IOV, not telco-hub)
- The AI must determine whether the CR is required or optional and what would break without it
- The AI must identify version-specific concerns (e.g., "this ICSP is deprecated in 4.19+, use IDMS instead")
- The AI must trace dependencies (e.g., "SriovNetwork requires SriovSubscription and SriovOperatorConfig")
- The AI must compare the CR against the reference to identify deviations or customizations

Before inspecting, read the reference skill at `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` for the CR catalogue, operator inventory, and version differences.

## Arguments

- `$1` (required): Path to a YAML file, a directory of CRs, or a CR kind name (e.g., `PerformanceProfile`, `SriovNetworkNodePolicy`).
- `--release` (optional): Target OpenShift release to check compatibility against (e.g., `4.19`). If omitted, checks against the latest release.

## Implementation

### 1. Read Reference Material

Read `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md`.

### 2. Identify the CR(s)

- If a file path: read the file and identify kind, apiVersion, name
- If a directory: scan all YAML files
- If a CR kind name: look up in the catalogue

### 3. Analyze Each CR

For each CR, determine:
1. **Use model**: Which use model(s) include this CR (core, hub, ran, or multiple)
2. **Category**: required or optional
3. **Operator**: Which operator owns this CR
4. **Dependencies**: What other CRs must exist for this to work (Namespace, OperatorGroup, Subscription, etc.)
5. **Release compatibility**: Which releases include this CR and any version-specific differences
6. **Deprecation status**: Whether this CR is deprecated or replaced in newer releases
7. **Customization notes**: What fields are typically customized vs left as reference defaults

### 4. Check Against Reference

If a file path was provided:
1. Compare field values against the reference CR for the target release
2. Identify deviations (custom values, missing fields, extra fields)
3. Flag potential issues (wrong API version, deprecated fields, missing labels)

### 5. Report

Output a structured analysis including:
- CR identification (kind, name, use model, category)
- Purpose and what it configures
- Dependency chain
- Release compatibility range
- Any deviations from the reference
- Recommendations

## Return Value

- **Format**: Structured text report
- **Content**: CR analysis with purpose, dependencies, compatibility, and recommendations

## Examples

### Example 1: Inspect a PerformanceProfile

```
/telco-reference:inspect PerformanceProfile --release 4.21
```

Reports that PerformanceProfile is a required CR in both telco-core and telco-ran, manages CPU isolation, hugepages, and RT kernel, depends on the Node Tuning Operator subscription, has separate x86_64/aarch64 variants since 4.20, and lists key customizable fields.

### Example 2: Inspect a directory of CRs

```
/telco-reference:inspect ./my-cluster-config/ --release 4.19
```

Scans all YAMLs, identifies each CR, reports which are from the reference and which are custom, flags any that use deprecated APIs (e.g., ICSP instead of IDMS for 4.19), and lists missing required CRs.

### Example 3: Inspect a specific file

```
/telco-reference:inspect telco-core/configuration/reference-crs/required/networking/sriov/sriovNetworkNodePolicy.yaml
```

Reports the SR-IOV policy's purpose, its relationship to SriovNetwork and SriovOperatorConfig, the available selector strategies (standard vs SetSelector), and any device-specific considerations.

## See Also

- `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md`
- `plugins/telco-reference/commands/create-config.md`
- `plugins/telco-reference/commands/upgrade-analysis.md`
