---
description: Parse RDS Analyzer reports and create ECOPS guidance tasks for section B items
argument-hint: "[--file <path.md>] [--partner <vendor> [RAN|CORE]] | [full RDS Analyzer report text]"
---

## Name
jira:analyze-rds-report

## Synopsis
```
/jira:analyze-rds-report [--file|-f <path>] [--partner|-p <vendor> [RAN|CORE]] [full RDS Analyzer report text]
```

## Description

The `jira:analyze-rds-report` command analyzes an RDS Analyzer report and applies strict business rules:

- It recognizes Section A (`The following deviations must be addressed:`) and Section B (`The following deviations require guidance from the telco team:`) separated by `==================================================`.
- It never creates Jira tickets for Section A and always returns the mandatory remediation statement.
- It creates ECOPS Task tickets for Section B:
  - One ticket per CR group in `1. Missing CRs:`
  - One ticket per CR block in `2. Diffs requiring review:`
- It enriches new tickets with:
  - Related ECOPS deviations (component `RDS Deviation`)
  - Inferred use-case labels (`ran`, `core`, `hub`) when determinable
  - OCP version (custom field when resolvable; label fallback)
  - Common Jira description archetype analysis and documentation
- Optionally (`--partner` / `-p`), it resolves the telco vendor (and RAN vs CORE when given) to a partner **ACCT** issue in Jira, documents that reference on each created ECOPS ticket, and adds a Jira **relates to** link between each new ECOPS issue and the ACCT issue.

This command is intended for operational triage where missing optional CR guidance and unresolved configuration diffs must be routed to the telco team.

## Implementation

### Step 1 - Collect report text and optional partner

1. Parse `$ARGUMENTS` for `--file <path>` or `-f <path>` (path may be quoted if it contains spaces).
2. Parse optional `--partner <value>` or `-p <value>`:
   - The value is everything after the flag until the next token that starts with `--` (or end of arguments). Examples: `--partner Nokia RAN`, `-p Ericsson CORE`, `--partner "Samsung RAN"`.
   - If the flag appears with no value, stop and list allowed partner forms (see **Step 1.6**).
3. If `--file` or `-f` is present:
   - Read the file from disk as UTF-8 text (treat a leading UTF-8 BOM as optional; strip only the BOM for processing, not other content).
   - If the path is missing, the file is not readable, or decoding fails, stop and report a clear file error. Do not create Jira issues.
   - Ignore any additional non-flag text in `$ARGUMENTS` when `--file` / `-f` supplies the report (the file is the sole source).
4. If no file flag: treat the remainder of `$ARGUMENTS` (after removing consumed flags and partner tokens) as the inline report body.
5. If there is no file and the inline body is empty, ask the user to paste the full RDS Analyzer report or pass `--file` / `-f` with a path to a Markdown (`.md`) file containing the report.
6. Preserve original whitespace after validation. Do not normalize indentation in diff content.

### Step 1.5 - Validate RDS Analyzer report shape (hard gate)

Before any parsing or Jira operations, confirm the input is a real RDS Analyzer report layout (the tool emits fixed headings and a separator). If validation fails, **stop immediately**: do not create issues, do not run Section B logic. Reply only with the error block in **Step 1.5.1** (you may add a one-line hint about the file path when `--file` was used).

Validation rules (all must pass):

1. The text contains the exact substring `The following deviations must be addressed:` (Section A header).
2. The text contains the exact substring `The following deviations require guidance from the telco team:` (Section B header).
3. The first occurrence of the Section A header starts **before** the first occurrence of the Section B header.
4. Between those two header positions (exclusive of the B header start), there is at least one line where the line equals `==================================================` after trimming trailing whitespace (the RDS Analyzer separator; exactly 50 `=` characters, no spaces).

If any rule fails, the content is not an RDS Analyzer report in the expected form (for example generic Markdown notes, partial paste, or another template).

#### Step 1.5.1 - Required user-facing message when validation fails

Output the following (replace only `{optional-path-hint}`; use an empty string if not using `--file`):

```
The input is not a valid RDS Analyzer report and was not processed.

RDS Analyzer exports a fixed structure. Your file or paste must include, in order:
1. A "must be addressed" section whose header is exactly: The following deviations must be addressed:
2. A separator line that contains only: ==================================================
3. A "require guidance" section whose header is exactly: The following deviations require guidance from the telco team:

Save or copy the full output from the RDS Analyzer tool (including those headings and the separator line) as Markdown if you use a .md file. {optional-path-hint}
```

### Step 1.6 - Resolve `--partner` to an ACCT issue (optional)

Run this after Step 1.5 passes. When `--partner` / `-p` is **not** set, skip this entire step and do not add ACCT links or Partner blocks on issues.

When it **is** set:

1. **Normalize** the user text: trim, collapse internal whitespace, compare **case-insensitively**.
2. **Match vendor and optional segment:** Known vendors are exactly: Nokia, Ericsson, Mavenir, Samsung, ZTE, Intel. The normalized string must start with one of these names (whole word). Optionally the same string may include a second whole word **`RAN`** or **`CORE`** (for example `nokia ran`, `ERICSSON CORE`, `Mavenir`). If extra tokens appear (for example `Nokia RAN Extra`), treat as invalid.
3. **Canonical label:** If `RAN` or `CORE` is present, use `{Vendor} RAN` or `{Vendor} CORE` (proper casing) for *Partner context*. If only the vendor appears, use `{Vendor}`.
4. If the value does not match any vendor or has invalid extra tokens, **stop before creating Jira issues** and respond with a short error listing allowed values: Nokia, Ericsson, Mavenir, Samsung, ZTE, Intel — each optional with `RAN` or `CORE` (examples: `Nokia RAN`, `ericsson core`).
5. **Resolve the ACCT issue by dynamic Jira search** (mandatory). Do **not** use a hardcoded vendor→key table and do **not** fall back to a default key if search is inconclusive.
   - **Browse base:** `https://redhat.atlassian.net/browse/{KEY}`.
   - **Search scope:** project **`ACCT`** only. Use the Jira issue search / JQL facilities available in the agent (MCP or equivalent).
   - **Queries (try in order, broaden only when the prior step returns zero issues):**
     1. Prefer matching **human-visible naming** to the **canonical label**. Start with a tight JQL such as phrase-style match on summary when supported, for example `project = ACCT AND summary ~ "\"Nokia RAN\""` — adjust quoting and operators to valid Jira Cloud JQL for your tool.
     2. If vendor-only: `project = ACCT AND summary ~ "Nokia"` (then rank carefully; see below).
     3. If still no candidates: broaden with `text ~` / `description ~` so **vendor** appears and, when the user specified a segment, **`RAN`** or **`CORE`** appears (both should align with the canonical label).
   - **Ranking:** Score candidates by how well **summary** (then **description**) matches the canonical label as a substring (case-insensitive). When segment was specified, strongly prefer issues whose naming clearly reflects **both** vendor and **RAN** or **CORE**. Deprioritize issues that only match one token or look unrelated.
   - **Selection (no fallback key):**
     - **One clear best match:** use that issue key for all subsequent steps.
     - **Zero matches after reasonable attempts:** stop before creating ECOPS issues; report failure, include the **canonical label** and the **JQL (or queries) tried**.
     - **Ambiguous** (tie between two or more, or no candidate clearly ahead): stop before creating ECOPS issues; list **top candidates** (issue key + summary) and ask the user to pick or refine `--partner`.
6. **Per new Section B ECOPS issue:** When building the **full** description for issue create, append the following block **after** the `h3. Description Pattern Analysis` section (or at the end of the description if pattern analysis is skipped):

```
h3. Partner account (ACCT)

*Partner context:* {Canonical partner label}
*ACCT reference:* [{ACCT-KEY}|https://redhat.atlassian.net/browse/{ACCT-KEY}]
```

7. **After** each such ECOPS issue is created, add a Jira issue link from the **new ECOPS issue** to the **resolved ACCT issue** with link type **relates to**.

8. In the **Step 7** final response, add **ACCT partner links**: for each new ECOPS key, the ACCT key (and URL) linked via relates to, and briefly note **which ACCT issue** was chosen and **why** (for example matched summary to canonical label).

### Step 2 - Parse required sections

1. Section A header (already validated):
   - `The following deviations must be addressed:`
2. Section B header (already validated):
   - `The following deviations require guidance from the telco team:`
3. Use `==================================================` separators to scope section boundaries.
4. If block-level content under Section B is malformed, record parsing errors in the final response (headers and separator are already guaranteed by Step 1.5).

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
   - Component: `RDS Deviation`
   - Summary: `RDS Guidance: Missing CR - {CR Group Name}`
3. Build description in this structure (keep spacing/blank lines exactly):

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

4. Infer metadata before create:
   - Use-case labels:
     - `ran` for terms like `ran`, `du`, `cu`, `near-rt`, `telco edge`
     - `core` for terms like `core`, `5gc`, `amf`, `smf`, `upf`
     - `hub` for terms like `hub`, `acm`, `mce`, `multicluster`
   - OCP version:
     - Detect from report tokens like `4.14`, `4.15`, `4.16`
     - Try setting ECOPS Target Version via project versions lookup and ID mapping
     - If ID mapping fails, add fallback label `ocp-<major>-<minor>` (for example `ocp-4-16`)
5. Create issue via MCP Jira create tool with:
   - `components=["RDS Deviation"]`
   - labels including `ai-generated-jira`, `rds-deviation`, inferred use-case labels, and optional OCP fallback label.
6. Run common-description analysis before create:
   - Build an analysis bundle from group name, CR name(s), template paths, and guidance text.
   - Invoke [skills/common-jira-descriptions/SKILL.md](../skills/common-jira-descriptions/SKILL.md) scoring method.
   - Capture:
     - primary pattern + confidence
     - secondary patterns (if any)
     - section checklist
     - top gaps
7. Append analysis documentation to issue description:

```
h3. Description Pattern Analysis

*Primary Pattern:* {Pattern Name} ({Confidence})
*Secondary Patterns:* {Pattern 2, Pattern 3 or N/A}

h4. Suggested Jira Sections
{code}
- {Section A}
- {Section B}
{code}

h4. Missing Information / Gaps
{code}
- {Gap A}
- {Gap B}
{code}
```

8. Discover related ECOPS deviation tickets and enrich:
   - Search ECOPS with JQL constrained to component `RDS Deviation` and deviation-specific terms derived from group/CR/template names.
   - Exclude the newly created issue key.
   - For each high-confidence match:
     - add issue link (`relates to`)
     - include in final response "Related tickets attached" list
9. If a related ticket appears impacting and has Release Note Text:
   - Treat as impacting when priority is `Blocker`/`Critical`, or labels/status include obvious impact markers.
   - If Release Note Text is present, post a comment on the new ticket:

```
h3. Related Impacting Deviation Context

Sourced from [<ISSUE-KEY>|<ISSUE-URL>] (component: RDS Deviation).

h4. Release Note Text
{quote}
{Release Note Text from related ticket}
{quote}
```

   - Add the source ticket key to "Propagation comments added" in the final response.

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
   - Component: `RDS Deviation`
   - Summary: `RDS Guidance: {Template Name}`
5. Build description in this structure (keep spacing/blank lines exactly):

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

6. Infer metadata before create:
   - Same use-case and OCP version logic as Step 4.
7. Run common-description analysis before create:
   - Build analysis bundle from template name, CR name, unresolved-difference text, and optional report context.
   - Invoke [skills/common-jira-descriptions/SKILL.md](../skills/common-jira-descriptions/SKILL.md) scoring method.
   - Capture primary/secondary patterns, checklist, and gaps.
8. Append analysis documentation to issue description using the same "Description Pattern Analysis" section format as Step 4.
9. Validate before create:
   - Template line exists and starts with `Template:`
   - CR line exists and starts with `CR Name:`
   - Diff body starts with `expected:` or `expected but not found:`
   - At least one diff section is extractable (`expected but not found` and/or `found but not expected`)
10. If validation fails for a block, skip creation for that block and append a parsing error entry for final reporting.
11. Create issue with component/labels/version metadata.
12. Run related-ticket discovery, linking, and optional Release Note Text propagation comment flow (same as Step 4).

### Step 5.1 - Diff section formatting rules

When building the Jira description for diff blocks:

1. Parse unresolved differences into named sections:
   - `expected but not found:`
   - `found but not expected:`
2. Render each present section in its own YAML code block.
3. Do not merge both sections into one code block.
4. Preserve exact indentation and markers (`>>`) inside each section body.
5. If only one section exists, include only that section heading and code block.

### Step 6 - Use the supporting skill

Follow [skills/rds-report-analyzer/SKILL.md](../skills/rds-report-analyzer/SKILL.md) for:

- Parsing strategy details
- Grouping logic for missing CRs
- Validation and error recording
- Optional `--partner` / `-p` resolution and ACCT **relates to** linking
- Related-ticket linking and release-note propagation comment rules
- Use-case label and OCP version inference rules
- Common Jira description analysis and pattern documentation rules
- Final response formatting rules

### Step 7 - Final response format

Return:

1. The mandatory Section A message (exact text).
2. Section B summary:
   - Number of Missing CR tickets created
   - Number of Diffs requiring review tickets created
   - List of created issue keys and links
   - Related tickets linked per new issue (if any)
   - Related-ticket Release Note Text comments added (if any)
   - Description pattern distribution (primary patterns across created tickets)
   - **ACCT partner links** (if `--partner` / `-p` was used): each new ECOPS issue and the ACCT issue key/URL linked with **relates to**
3. Parsing errors encountered (if any), so the user can manually review those report blocks.

## Arguments

- **`--file` / `-f`**: Path to a Markdown (`.md`) file whose body is the full RDS Analyzer report (same text you would paste inline).
- **`--partner` / `-p`**: Optional telco vendor and segment for ACCT linking. Accepts vendor alone (`Nokia`, `zte`) or vendor plus `RAN` / `CORE` (case-insensitive, flexible spacing). The ACCT issue is **found by Jira search** in project `ACCT` matching that naming (see Step 1.6); there is no static key table.
- **Free text report**: Full RDS Analyzer report body including section headings and separators, when no file flag is used.

## Return Value

- **Claude agent text** with:
  - Mandatory Section A no-ticket statement
  - Ticket creation counts by subsection
  - Created issue keys + links
  - Common description patterns documented per issue
  - ACCT partner link summary when `--partner` / `-p` was set
  - Parsing/validation errors, if present

## Examples

```
/jira:analyze-rds-report --file ./cluster-rds-report.md --partner "Nokia RAN"
```

```
/jira:analyze-rds-report --file ./cluster-rds-report.md
```

```
/jira:analyze-rds-report The following deviations must be addressed:
...
==================================================
The following deviations require guidance from the telco team:
...
```
