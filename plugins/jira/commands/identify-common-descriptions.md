---
description: Given rough text or a Jira key, identify which common Jira description patterns apply and outline sections, gaps, and examples
argument-hint: "[free-text notes] | <ISSUE-KEY>"
---

## Name
jira:identify-common-descriptions

## Synopsis
```
/jira:identify-common-descriptions [free-text notes]
/jira:identify-common-descriptions <ISSUE-KEY>
```

## Description

The `jira:identify-common-descriptions` command uses **AI pattern matching** over the user’s **input text** and/or a **Jira issue’s** existing fields to answer:

- Which **common Jira description archetypes** (incident, CI regression, upgrade failure, functional bug, performance, security, docs, RFE, tech debt, operator/OLM, networking, storage, etc.) best fit the content.
- Which **description sections** should appear for those patterns and what is **still missing** from the input.
- A concise **skeleton** the user can paste into Jira, without inventing facts not present in the source material.

This is not a generic summarizer: it applies the **taxonomy and scoring methodology** in the skill so outputs stay consistent and actionable for engineering Jira workflows.

## Implementation

### Step 1 — Resolve input

- If `$ARGUMENTS` looks like an **issue key** (e.g. `OCPBUGS-12345`, `PROJECT-1`): treat as **issue mode**.
- Otherwise treat as **text mode** using the full argument string as the body of analysis. If empty, ask for text or a key.

### Step 2 — Issue mode (optional)

When in issue mode:

1. Fetch the issue via MCP (fields at minimum: `summary`, `description`, `issuetype`, `labels`, `components`, `priority`).

```text
mcp__atlassian__jira_get_issue(issue_key="<key>", fields="summary,description,issuetype,labels,components,priority")
```

2. Build an **analysis bundle**: summary + description + issue type name + labels + components as plain text for the skill.
3. If fetch fails, report the error and fall back to asking for pasted content.

### Step 3 — Invoke the skill

Load and follow [skills/common-jira-descriptions/SKILL.md](../skills/common-jira-descriptions/SKILL.md):

- Score archetypes from **signals** in the skill.
- Return top patterns, **evidence** from input (quotes or paraphrase tied to source), **section checklist**, and **gaps**.
- Optionally produce a **bullet skeleton** for the strongest pattern; use `TBD` or explicit questions for unknowns—never fabricate versions, URLs, or test results.

### Step 4 — Cross-link when helpful

If the primary pattern is clearly a **bug**, point to `/jira:create bug` and the create-bug skill for full template. If **story/RFE**, point to create-story / create-feature flows as appropriate.

## Arguments

- **Free text**: Notes, slack paste, partial description—any natural language.
- **Issue key**: Single key to fetch and classify.

## Examples

```
/jira:identify-common-descriptions prow job aws-serial failing after merge 12345 on 4.16 nightly
```

```
/jira:identify-common-descriptions OCPBUGS-99999
```

## Return Value

- **Claude agent text**: Pattern name(s), confidence, evidence, recommended sections, gaps, and optional skeleton—per the skill’s return shape.
