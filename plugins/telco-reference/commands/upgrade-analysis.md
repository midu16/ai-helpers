---
description: Analyze migration and upgrade path between two OpenShift telco reference releases
argument-hint: "<from-release> <to-release> [--use-model <model>] [--detailed]"
---

## Name
telco-reference:upgrade-analysis

## Synopsis
```
/telco-reference:upgrade-analysis <from-release> <to-release> [--use-model <model>] [--detailed]
```

## Description

The `telco-reference:upgrade-analysis` command provides a comprehensive analysis of what changes between two OpenShift release branches in the telco-reference repository, identifying breaking changes, new features, deprecated CRs, migration steps, and upgrade risks.

This requires AI reasoning because:
- The AI must identify which changes are breaking vs additive and assess the risk level
- The AI must determine the correct migration order (e.g., "upgrade OCP first, then OLM operators, then apply new CRs")
- The AI must trace dependency chains across changes (e.g., "ICSP removal requires IDMS creation before upgrading")
- The AI must identify changes that affect PolicyGenerator references, kube-compare baselines, or ArgoCD applications
- The AI must provide actionable remediation steps, not just a diff listing
- For multi-version jumps (e.g., 4.16 → 4.21), the AI must aggregate and order all intermediate changes

Before analyzing, read the reference skill at `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` for the version differences, breaking changes, and operator timeline.

## Arguments

- `$1` (required): Source release — e.g., `4.16`, `4.19`.
- `$2` (required): Target release — e.g., `4.20`, `4.21`, `main`.
- `--use-model` (optional): Restrict analysis to a specific use model (`core`, `hub`, `ran`, or `all`). Default: `all`.
- `--detailed` (optional): Include per-CR diff details and full migration steps.

## Implementation

### 1. Read Reference Material

Read `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md` to load the version differences and breaking changes.

### 2. Validate Versions

1. Confirm both releases exist (4.14–4.21, main)
2. Confirm from-release < to-release
3. Identify all intermediate versions in the path

### 3. Collect Changes

For each version step in the upgrade path:

1. **Added CRs**: New files that appear in the target but not the source
2. **Removed CRs**: Files that exist in the source but not the target
3. **Modified CRs**: Files that exist in both but have changed content
4. **Breaking changes**: Changes that require manual intervention (API changes, directory restructuring, deprecated fields)
5. **New use models**: If a use model becomes available (hub at 4.18, ran at 4.19)

### 4. Classify Changes

For each change, classify:

| Risk Level | Criteria |
|------------|----------|
| **Critical** | Breaking API change, removed CR without replacement, directory restructuring affecting PolicyGenerator |
| **High** | Deprecated API still functional but must migrate soon, new required operator |
| **Medium** | New optional CRs available, kube-compare reference updates |
| **Low** | Documentation changes, Tekton pipeline updates, cosmetic changes |

### 5. Generate Migration Plan

Produce an ordered migration plan:

1. **Pre-upgrade preparations** (backup, validate current state, create new CRs that must exist before upgrade)
2. **Platform upgrade** (OCP version via ClusterVersion or IBU)
3. **OLM operator upgrades** (Subscription channel changes)
4. **Post-upgrade CR changes** (add new CRs, remove deprecated ones, update paths)
5. **Validation** (kube-compare check, operator status verification)

### 6. Report

Output a structured upgrade analysis including:
- Executive summary (risk level, estimated effort)
- Version-by-version changelog
- Breaking changes with remediation
- New features available after upgrade
- Complete migration checklist
- Rollback considerations

## Return Value

- **Format**: Structured text report with tables and checklists
- **Content**: Upgrade analysis with risk assessment, migration steps, and validation plan

## Examples

### Example 1: Single-version core upgrade

```
/telco-reference:upgrade-analysis 4.20 4.21 --use-model core
```

Reports: cert-manager CRs available as optional, TunedPerformancePatch added to kube-compare, core-upgrade-finish already available. Risk: Low — no breaking changes for core between 4.20 and 4.21.

### Example 2: Multi-version jump

```
/telco-reference:upgrade-analysis 4.16 4.21 --use-model core --detailed
```

Reports 5 version jumps with cumulative changes: PolicyGenerator adoption (4.17), CLO v5→v6 (4.18), ICSP→IDMS (4.19), infrastructure CR + subscription-validator (4.20), cert-manager (4.21). Risk: High — multiple breaking changes requiring ordered migration.

### Example 3: Full fleet analysis

```
/telco-reference:upgrade-analysis 4.19 4.21
```

Analyzes all three use models: core (ICSP→IDMS already done, add cert-manager, add core-upgrade-finish), hub (add kube-compare, add cert-manager policies, add observabilityRoutePolicy), ran (source-crs restructuring in 4.20, SiteConfig→ClusterInstance in 4.21, add PTP T-BC/T-TSC). Risk: High — RAN directory restructuring and SiteConfig migration are breaking.

## See Also

- `plugins/telco-reference/skills/telco-reference-catalogue/SKILL.md`
- `plugins/telco-reference/commands/feature-summary.md`
- `plugins/telco-reference/commands/create-config.md`
