---
description: Summarize features, operators, and capabilities available in a telco reference release branch
argument-hint: "<release> [--use-model <model>] [--format <format>]"
---

## Name
telco-reference:feature-summary

## Synopsis
```
/telco-reference:feature-summary <release> [--use-model <model>] [--format <format>]
```

## Description

The `telco-reference:feature-summary` command produces a comprehensive summary of all features, operators, CRs, and capabilities available in a specific OpenShift telco reference release branch. It provides a clear picture of what a given release supports for planning, documentation, and decision-making.

This requires AI reasoning because:
- The AI must aggregate information across all three use models and categorize by function (networking, storage, performance, management, etc.)
- The AI must distinguish between required and optional components and explain the trade-offs
- The AI must identify what's new in this release compared to the previous one
- The AI must note any prerequisites, limitations, or known issues for each feature
- The AI must provide architectural context (e.g., how PolicyGenerator, ArgoCD, and TALM work together)

Before summarizing, read the reference skill at `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` for the complete CR catalogue and operator inventory.

## Arguments

- `$1` (required): OpenShift release — e.g., `4.19`, `4.21`, `main`.
- `--use-model` (optional): Restrict summary to a specific use model (`core`, `hub`, `ran`). Default: all available models for the release.
- `--format` (optional): Output format — `summary` (default), `table`, or `detailed`.

## Implementation

### 1. Read Reference Material

Read `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md`.

### 2. Determine Scope

1. Identify which use models are available on the target release
2. If a specific use model is requested, validate it exists on that release

### 3. Compile Feature Inventory

For each available use model, compile:

1. **Operators**: List all required and optional operators with their purpose
2. **CRs**: Count and categorize reference CRs (required vs optional, by operator)
3. **Capabilities**: High-level capabilities enabled (e.g., "CPU isolation", "PTP timing", "fleet management")
4. **Architecture**: Deployment patterns (PolicyGenerator, ArgoCD, ZTP, kube-compare)
5. **Install method**: How clusters are provisioned (SiteGen/MCE, ABI, ZTP/ACM)
6. **Upgrade strategy**: Available upgrade mechanisms (TALM CGU, IBU, manual)

### 4. Identify What's New

Compare against the previous release to highlight:
- New operators or features introduced in this release
- Changed or improved capabilities
- Deprecated features being removed

### 5. Generate Summary

Produce a structured summary organized by:
1. Release overview (available use models, total CRs, key highlights)
2. Per-use-model breakdown (operators, CRs, capabilities)
3. What's new in this release
4. Known limitations or prerequisites
5. Supported topologies (SNO, 3-node, standard, HA)

## Return Value

- **Format**: Structured text summary, table, or detailed report
- **Content**: Feature inventory with operator list, CR counts, capabilities, and release highlights

## Examples

### Example 1: Full release summary

```
/telco-reference:feature-summary 4.21
```

Produces a comprehensive summary of all three use models for 4.21: telco-core (170 CRs, 12 operators), telco-hub (194 CRs, 8 operators), telco-ran (301 CRs, 15 operators). Highlights cert-manager addition, PTP T-BC/T-TSC configs, and SiteConfig→ClusterInstance migration.

### Example 2: RAN-specific summary

```
/telco-reference:feature-summary 4.19 --use-model ran
```

Summarizes the initial telco-ran release: 297 CRs across 15 operator categories, SNO/3-node/standard topologies, ZTP via ACM+ArgoCD, IBU support via LCA+OADP, 18 PTP configurations, and x86_64 PerformanceProfiles.

### Example 3: Table format

```
/telco-reference:feature-summary 4.20 --format table
```

Outputs a concise table listing each operator, its status (required/optional), available CRs, and the use models it applies to.

## See Also

- `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md`
- `plugins/telco-reference/commands/upgrade-analysis.md`
- `plugins/telco-reference/commands/create-config.md`
