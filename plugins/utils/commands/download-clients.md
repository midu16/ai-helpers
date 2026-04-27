---
description: OpenShift Client Download — interpret natural-language CLI needs, resolve official downloads for OpenShift and related tools, verify integrity, and install binaries to a user-approved prefix
argument-hint: "[free-text describing which clients and versions]"
---

## Name
utils:download-clients

## Synopsis
```
/utils:download-clients [free-text describing which clients and versions]
```

## Description

The `utils:download-clients` command implements **OpenShift Client Download**: it turns an informal description (“I need oc and openshift-install for OpenShift 4.16 on Linux”, “latest ROSA CLI and helm 3”) into a **concrete download plan**, then performs **official-source** retrieval with **checksum verification when published**, and installs binaries to a **prefix the user agrees to** (typically a user-writable directory).

This command is **not** a dumb wrapper around a fixed script: it requires the agent to **interpret intent**, **disambiguate versions and architecture**, **pick correct mirror paths**, and **explain trade-offs** (for example, stable versus a specific z-stream). It complements team-specific bootstrap scripts by encoding **vendor rules**, **verification**, and **PATH** guidance in a repeatable way.

**Load the skill first**: follow `plugins/utils/skills/openshift-client-download/SKILL.md` (**OpenShift Client Download**) for the full catalog, URL patterns, verification steps, and safety notes.

## Implementation

### 1. Ingest the user description

- Parse `$ARGUMENTS` (or the user’s follow-up text) for: desired **tools**, **version constraints**, **OS/arch** hints, and **install location** preferences.
- If critical information is missing (for example “4.16” without patch when they need installer/media-matching bits), ask **one** focused question before large downloads.

### 2. Discover platform

Run (or equivalent):

```bash
uname -s
uname -m
```

Map to vendor naming (`amd64`/`arm64`, archive names) as described in the **OpenShift Client Download** skill.

### 3. Build the artifact plan

Using the **OpenShift Client Download** skill’s default catalog (OpenShift mirror clients, Helm, ROSA, `opm`, `operator-sdk`, optional helpers):

- Resolve **directory or release** for each component.
- Choose **exact filenames** from the vendor listing for that version—do not guess patch numbers.
- Note **checksum files** shipped alongside artifacts (for example `sha256sum.txt` on the OpenShift mirror).

### 4. Stage, verify, install

- Use a disposable staging directory under `.work/openshift-client-download/` (gitignored) unless the user specifies another **absolute** path.
- Download with failure-on-HTTP-error (`curl -fL` or equivalent).
- Verify SHA256 when the vendor provides it; record **verified** / **not available** per artifact.
- Extract, copy intended binaries to the agreed prefix, `chmod +x`, and avoid leaving stale partial downloads without mentioning them.

### 5. Report outcomes

Return:

- What was installed, full paths, and **export PATH=...** (or shell profile snippet) appropriate to their shell.
- Suggested smoke commands (`oc version --client`, etc.).
- Anything skipped or deferred, with the **next** user action.

## Arguments

- **Free text** (optional but typical): Natural-language list of clients and version intent. If omitted, ask what they need and which OpenShift version (if any) applies.

## Examples

1. **OpenShift clients for a specific release**:
   ```
   /utils:download-clients oc and openshift-install for OpenShift 4.16 on Linux x86_64 into ~/bin
   ```

2. **Helpers only**:
   ```
   /utils:download-clients helm 3 and yq latest for mac arm64
   ```

3. **Ambiguous input** (agent should clarify before downloading):
   ```
   /utils:download-clients openshift 4.15 clients
   ```
   Ask whether they need **only** `oc`/`kubectl` or also **`openshift-install`**, and confirm patch or “latest 4.15.z on mirror”.

## Return Value

- **Claude agent text**: Concise **OpenShift Client Download** summary of resolved versions, URLs or mirror directories, checksum verification status, installed paths, PATH updates, and smoke-test results or errors.
