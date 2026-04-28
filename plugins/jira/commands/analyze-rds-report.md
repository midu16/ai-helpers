---
description: Parse RDS Analyzer reports and create ECOPS guidance tasks for section B items
argument-hint: "[full RDS Analyzer report text]"
---

## Name
jira:analyze-rds-report

## Synopsis
```
/jira:analyze-rds-report [full RDS Analyzer report text]
```

## Description

The `jira:analyze-rds-report` command analyzes an RDS Analyzer report and applies strict business rules:

- It recognizes Section A (`The following deviations must be addressed:`) and Section B (`The following deviations require guidance from the telco team:`) separated by `==================================================`.
- It never creates Jira tickets for Section A and always returns the mandatory remediation statement.
- It creates ECOPS Task tickets for Section B:
  - One ticket per CR group in `1. Missing CRs:`
  - One ticket per CR block in `2. Diffs requiring review:`

This command is intended for operational triage where missing optional CR guidance and unresolved configuration diffs must be routed to the telco team.

## Implementation

### Step 1 - Collect report text

1. Use `$ARGUMENTS` as the report body when provided.
2. If empty, ask the user to paste the full RDS Analyzer report.
3. Preserve original whitespace. Do not normalize indentation in diff content.

### Step 2 - Parse required sections

1. Identify Section A header:
   - `The following deviations must be addressed:`
2. Identify Section B header:
   - `The following deviations require guidance from the telco team:`
3. Use `==================================================` separators to scope section boundaries.
4. If either section cannot be located, continue with what is present and report parsing errors in the final response.

### Step 3 - Section A handling (no tickets)

For Section A content:

- Do not create Jira issues.
- Always include this exact statement in the final response:

> The deviations listed under 'Must be addressed' do not require Jira tickets. These deviations represent configuration gaps that must be remediated directly by the partner or customer to align with the reference configuration.

### Step 4 - Section B / Missing CRs

In Section B, find subsection:

- `1. Missing CRs:`

Rules:

1. Parse CR entries and group by CR configuration group name (for example `kdump-configuration`).
2. Create one Jira ticket per group using:
   - Project: `ECOPS`
   - Issue Type: `Task`
   - Summary: `RDS Guidance: Missing CR - {CR Group Name}`
3. Build description exactly in this structure:

```
h3. RDS Analyzer Report - Missing Optional CR

*Group:* {Group Name}
*CR Name:* {CR Name}

h3. Missing Templates
{code}
{List of missing template paths}
{code}

h3. Guidance Required
While marked as optional, these CRs are expected in most clusters. Please clarify why they are not being used.
```

4. Create issue via MCP Jira create tool.

### Step 5 - Section B / Diffs requiring review

In Section B, find subsection:

- `2. Diffs requiring review:`

Rules:

1. Split CR diff blocks using:
   - `--------------------------------------------------`
2. For each block, extract:
   - `Template Name` from `Template: ...`
   - `CR Name` from `CR Name: ...`
   - Diff details from `Unresolved differences:` through end of block
3. Preserve diff text exactly:
   - Keep all `>>` markers
   - Keep indentation and blank lines
   - Keep complete expected/found sections
   - Keep `expected but not found:` lines when present
4. Create one Jira ticket per CR block using:
   - Project: `ECOPS`
   - Issue Type: `Task`
   - Summary: `RDS Guidance: {Template Name}`
5. Build description exactly in this structure:

```
h3. RDS Analyzer Report Data

*Template:* {Template Name}
*CR Name:* {CR Name}

h3. Unresolved Differences
{code:yaml}
{Complete Diff Details}
{code}
```

6. Validate before create:
   - Template line exists and starts with `Template:`
   - CR line exists and starts with `CR Name:`
   - Diff body starts with `expected:` or `expected but not found:`
7. If validation fails for a block, skip creation for that block and append a parsing error entry for final reporting.

### Step 6 - Use the supporting skill

Follow [skills/rds-report-analyzer/SKILL.md](../skills/rds-report-analyzer/SKILL.md) for:

- Parsing strategy details
- Grouping logic for missing CRs
- Validation and error recording
- Final response formatting rules

### Step 7 - Final response format

Return:

1. The mandatory Section A message (exact text).
2. Section B summary:
   - Number of Missing CR tickets created
   - Number of Diffs requiring review tickets created
   - List of created issue keys and links
3. Parsing errors encountered (if any), so the user can manually review those report blocks.

## Arguments

- **Free text report**: Full RDS Analyzer report body including section headings and separators.

## Return Value

- **Claude agent text** with:
  - Mandatory Section A no-ticket statement
  - Ticket creation counts by subsection
  - Created issue keys + links
  - Parsing/validation errors, if present

## Examples

```
/jira:analyze-rds-report The following deviations must be addressed:
...
==================================================
The following deviations require guidance from the telco team:
...
```
