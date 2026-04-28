---
name: RDS Report Analyzer
description: Parse RDS Analyzer reports and create ECOPS guidance tasks for section B while enforcing no-ticket handling for section A
---

# RDS Report Analyzer

This skill is the implementation guide for `/jira:analyze-rds-report`.

## Agent Identity and Goal

You are the **RDS Report Analyzer**. Process RDS Analyzer reports, parse the two required sections, and create Jira tickets only for guidance-required items according to the business rules below.

## Required Input Structure

The report is expected to include these section headers:

1. `The following deviations must be addressed:`
2. `The following deviations require guidance from the telco team:`

Sections are separated by:

- `==================================================`

If delimiters or headers are malformed, continue best-effort parsing and log parsing errors.

## Rule 1 - Section A (Must be addressed)

Action:

- Never create Jira tickets for Section A items.

Mandatory final-response statement (exact text):

`The deviations listed under 'Must be addressed' do not require Jira tickets. These deviations represent configuration gaps that must be remediated directly by the partner or customer to align with the reference configuration.`

## Rule 2 - Section B (Require guidance)

Section B may contain both subsections below. Handle each independently.

### 2a - Missing CRs

Identification:

- Locate subsection starting with `1. Missing CRs:`

Behavior:

1. Parse listed optional CR entries and template paths.
2. Group entries by CR group name (for example `kdump-configuration`, `mount-namespace-configuration`).
3. Create one Jira issue per group.

Ticket fields:

- Project: `ECOPS`
- Issue type: `Task`
- Component: `RDS Deviation`
- Summary: `RDS Guidance: Missing CR - {CR Group Name}`
- Description template (use exact spacing to avoid formatting issues):

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

### 2a.1 - Metadata inference (labels + version)

Before creating each issue:

1. Infer use-case labels from report signals:
   - `ran`: `ran`, `du`, `cu`, `near-rt`, `telco edge`
   - `core`: `core`, `5gc`, `amf`, `smf`, `upf`
   - `hub`: `hub`, `acm`, `mce`, `multicluster`
2. Always include baseline labels: `ai-generated-jira`, `rds-deviation`
3. Detect OCP version hints (for example `4.14`, `4.15`, `4.16`) from the report.
4. If a version is found:
   - Attempt to resolve a matching ECOPS Target Version using project versions and version IDs.
   - If a valid version ID cannot be resolved, add fallback label `ocp-<major>-<minor>` (for example `ocp-4-16`).

### 2a.2 - Related deviation discovery and propagation

After creating each issue:

1. Search for related ECOPS issues:
   - Project `ECOPS`
   - Component `RDS Deviation`
   - Text similarity based on deviation fingerprint terms (group name, CR name, template name when present)
   - Exclude the newly created issue
2. For each strong match, add issue link (`relates to`) between new and existing ticket.
3. Determine if a related ticket is impacting (best effort):
   - Priority `Blocker`/`Critical`, or clear impact labels/status markers.
4. If related ticket is impacting and has Release Note Text:
   - Add a comment to the new ticket with the related ticket reference and quoted Release Note Text.
5. Record all linked related keys and all propagation-comment source keys for final reporting.

### 2b - Diffs requiring review

Identification:

- Locate subsection starting with `2. Diffs requiring review:`

Block parsing:

1. Split with separator `--------------------------------------------------`.
2. For each block:
   - Extract template name from line starting with `Template:`
   - Extract CR name from line starting with `CR Name:`
   - Extract unresolved diff details from `Unresolved differences:` to end of block

Diff preservation requirements (strict):

- Preserve all lines exactly as in input
- Keep `>>` markers
- Keep indentation and blank lines
- Keep complete expected/found content
- Keep `expected but not found:` when present
- Do not summarize or truncate

Ticket fields:

- Project: `ECOPS`
- Issue type: `Task`
- Component: `RDS Deviation`
- Summary: `RDS Guidance: {Template Name}`
- Description template (use exact spacing to avoid formatting issues):

```
h3. RDS Analyzer Report Data

*Template:* {Template Name}
*CR Name:* {CR Name}

h3. Unresolved Differences
h4. expected but not found
{code:yaml}
{Expected-but-not-found content only}
{code}

h4. found but not expected
{code:yaml}
{Found-but-not-expected content only}
{code}
```

Diff presentation rules:

1. Parse and separate unresolved diff into section buckets:
   - `expected but not found:`
   - `found but not expected:`
2. Render each existing bucket as its own `{code:yaml}` block.
3. Never put both buckets in a single code block.
4. Preserve original indentation, blank lines, and `>>` markers inside each bucket.
5. If only one bucket exists in the source, output only that bucket heading + code block.

### 2b.1 - Metadata inference (labels + version)

Apply the same label/version inference as section 2a before creating each diff ticket.

### 2b.2 - Related deviation discovery and propagation

Apply the same related-ticket linking and Release Note Text propagation flow as section 2a after each diff ticket is created.

## Validation Rules

For each diff block, validate:

1. Template name extracted from a line with `Template:`
2. CR name extracted from a line with `CR Name:`
3. Diff body starts with `expected:` or `expected but not found:`
4. At least one diff bucket is extractable (`expected but not found` and/or `found but not expected`)

On validation failure:

- Do not create a ticket for that block
- Add a parsing error describing which block failed and why

## Jira creation details

Create each issue with:

- `project_key="ECOPS"`
- `issue_type="Task"`
- `components=["RDS Deviation"]`
- labels: baseline + inferred use-case labels + optional OCP fallback label
- target version custom field when version ID is confidently resolved

For related-ticket enrichment:

- Use issue search to find candidate related deviations
- Use issue link creation with `relates to`
- Use issue comment creation for propagated Release Note Text context

## Jira Creation

Use Jira issue creation MCP tool for each issue:

- One ticket per Missing CR group
- One ticket per Diffs block that passes validation

Capture created issue key and link for final summary.

## Final Response Contract

Always return:

1. Mandatory Section A statement (exact text)
2. Ticket summary for Section B:
   - Count of Missing CR tickets created
   - Count of Diffs requiring review tickets created
   - List of issue keys and links
   - Related tickets linked per created issue (if any)
   - Release Note Text propagation comments added (if any)
3. Parsing errors, if any, so user can manually review report content

If no Section B items were parsed, still return the mandatory Section A statement and explicitly note that no ECOPS tickets were created.
