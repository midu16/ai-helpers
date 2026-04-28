---
name: Common Jira Description Patterns
description: Map free-text notes or an existing issue into common Jira description archetypes with section outlines, recognition signals, and OpenShift-oriented examples
---

# Common Jira Description Patterns

Use this skill when the user (or `/jira:identify-common-descriptions`) needs to **recognize which kind of Jira description** their input resembles and how to **shape** summary, body, and acceptance-style sections—without duplicating full bug/story authoring guides (see [Create Jira Bug](../create-bug/SKILL.md) and [Create Jira Story](../create-story/SKILL.md) for end-to-end templates).

## When to Use

- The user pastes **rough notes**, **chat snippets**, or **half-written** text and wants to know the **closest common patterns** and a **skeleton description**.
- The user gives an **issue key**; after fetching summary/description via MCP, classify and suggest **improvements** or **missing sections**.
- Triaging: “what bucket is this?” before picking issue type (Bug vs Story vs Task) or linking to the right template.

## Inputs

1. **Free text** — symptoms, context, links, partial bullets.
2. **Issue key** (optional) — agent fetches `summary`, `description`, `issuetype`, `labels`, `components` via MCP, then merges with any extra text the user provided.

If neither text nor fetchable issue exists, ask for a short paste or a key.

## Method: score archetypes, then outline

1. **Normalize input**: lowercasing for matching only; preserve originals for quoted output.
2. **Score** each archetype below using **signals** (keywords, phrases, structures). Allow **multiple** top patterns (e.g. CI regression + upgrade) when evidence supports it; rank by strength.
3. **Report** for the top 1–3 patterns:
   - **Pattern name** and one-line **definition**
   - **Why it matches** (specific phrases or facts from input—no fabrication)
   - **Recommended sections** (ordered list)
   - **Gaps**: what the user should still add (versions, must-gather, PR links, etc.)
4. **Optional**: Draft a **merged outline** (bullet skeleton) combining the strongest pattern’s sections; do not invent technical claims not present in input.

## Archetypes and signals

### 1. Customer / production incident

**Signals:** `customer`, `production`, `outage`, `SEV`, `escalation`, `down`, `impact`, `SRE`, `pager`, time pressure, business impact.

**Sections:** Impact / scope · Timeline · Current mitigation · Environment (version, topology) · Reproduction or observability · Ask (engineering vs support).

### 2. CI / test regression

**Signals:** `prow`, `CI`, `e2e`, `flake`, `job`, `test failed`, `payload`, `rehearse`, `pull request`, link to prow.ci or deck.

**Sections:** Failing job name + link · Pass/fail history · Payload or branch · Minimal repro or suspected commit · Artifacts (must-gather, junit).

### 3. Upgrade / migration failure

**Signals:** `upgrade`, `4.x to 4.y`, `CVO`, `ClusterVersion`, `blocked`, `pre-upgrade`, `post-upgrade`, `etcd`, `migration`.

**Sections:** From → to versions · CVO conditions / operator degraded · Steps taken · Cluster state · Logs references.

### 4. Functional bug (product behavior)

**Signals:** `expected`, `actual`, `wrong`, `incorrect`, `regression` (behavioral), component names without “CI only”.

**Sections:** Problem · Version · Repro steps · Actual vs expected · Frequency · Workaround if any.

### 5. Performance / scale / resource

**Signals:** `slow`, `latency`, `CPU`, `memory`, `OOM`, `throttle`, `scale`, `throughput`, `p99`, profiling.

**Sections:** Workload · Metrics / graphs · Baseline vs observed · Node pool / size · Data collection window.

### 6. Security / compliance

**Signals:** `CVE`, `vulnerability`, `CVSS`, `SAST`, `compliance`, `CVE-`, `weakness`, `disclosure`.

**Sections:** Affected component · CVE id · Exposure assessment · Patched versions · References (advisory).

### 7. Documentation / process / “how do I”

**Signals:** `docs`, `documentation`, `unclear`, `missing steps`, `runbook`, `how to`, no broken product behavior.

**Sections:** Doc URL · What is wrong or missing · Suggested fix · Audience.

### 8. Enhancement / RFE (new capability)

**Signals:** `feature`, `would like`, `enhancement`, `RFE`, `support for`, `add ability`, user story tone without failure.

**Sections:** Problem / use case · Proposed behavior · Acceptance criteria · Out of scope · Dependencies.

### 9. Tech debt / refactor / internal cleanup

**Signals:** `refactor`, `cleanup`, `deprecated`, `remove dead code`, `no user-facing`, `maintainability`.

**Sections:** Current pain · Proposed change · Risk · Test plan · Rollout.

### 10. Operator / OLM / install path

**Signals:** `operator`, `OLM`, `Subscription`, `InstallPlan`, `CSV`, `bundle`, `catalog`.

**Sections:** Operator name/version · Subscription spec · Events from install namespace · Expected vs observed CSV state.

### 11. Networking (DNS, ingress, SDN/OVN)

**Signals:** `DNS`, `ingress`, `route`, `NetworkPolicy`, `OVN`, `CNO`, `packet drop`, `connection refused`.

**Sections:** Topology · Repro network path · `oc` commands run · Captures (tcpdump, must-gather network) if mentioned.

### 12. Storage / PVC / CSI

**Signals:** `PVC`, `PV`, `CSI`, `mount`, `volume`, `snapshot`, `provision`.

**Sections:** Storage class · Claim spec · Events · Node logs reference if provided.

## Confusion handling

- **CI vs product bug:** CI keywords + job links → lean regression; user-only functional wrongness without CI → lean functional bug.
- **Incident vs bug:** time-critical widespread customer pain → incident pattern even if also a bug.
- **Story vs task:** user-facing outcome → enhancement/story; internal-only → tech debt/task.

## References

- Wiki markup: [reference/wiki-markup.md](../../reference/wiki-markup.md)
- MCP fields: [reference/mcp-tools.md](../../reference/mcp-tools.md)

## Return shape (for the agent)

Emit a short, scannable response:

1. **Primary pattern** (and confidence: high / medium / low)
2. **Secondary patterns** if any
3. **Section checklist** for the primary
4. **Suggested next sentences** or bullets only where grounded in user input; otherwise list **placeholders** for missing facts
