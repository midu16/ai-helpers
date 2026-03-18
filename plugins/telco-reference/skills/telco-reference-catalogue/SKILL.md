---
name: Telco Reference Catalogue
description: |
  Comprehensive reference for OpenShift telco reference configurations across
  all release branches (4.14–4.21 and main). Covers telco-core, telco-hub, and
  telco-ran use models, with operator inventories, CR catalogues, version
  differences, upgrade paths, and deployment architecture.

  Triggers: "telco reference", "telco core", "telco hub", "telco ran",
  "reference CR", "source CR", "kube-compare", "PolicyGenerator",
  "upgrade path", "release comparison", "feature summary", "RDS",
  "RAN profile", "DU profile", "SNO", "telco config"
---

# Telco Reference Catalogue

Complete reference material for the
[openshift-kni/telco-reference](https://github.com/openshift-kni/telco-reference)
repository, which provides validated reference configurations for three OpenShift
telco use models across all supported release branches.

## Repository Layout

```
telco-reference/
├── telco-core/          # Telco Core (large cluster, vDU/vCU workloads)
│   ├── configuration/
│   │   ├── reference-crs/           # Reference CRs (required/ + optional/)
│   │   ├── reference-crs-kube-compare/  # Templated CRs for kube-compare
│   │   ├── template-values/         # ConfigMap values for PolicyGenerator
│   │   ├── core-baseline.yaml       # PolicyGenerator baseline
│   │   ├── core-overlay.yaml        # PolicyGenerator overlay
│   │   ├── core-upgrade.yaml        # Upgrade PolicyGenerator
│   │   └── core-upgrade-finish.yaml # Post-upgrade PolicyGenerator
│   └── install/                     # SiteConfig / ClusterInstance artifacts
├── telco-hub/           # Telco Hub (management cluster)
│   ├── configuration/
│   │   ├── reference-crs/           # Reference CRs (required/ + optional/)
│   │   ├── reference-crs-kube-compare/
│   │   ├── example-overlays-config/ # Kustomize overlay examples
│   │   └── kustomization.yaml       # ArgoCD-driven config
│   ├── install/                     # ABI install artifacts
│   └── scripts/
└── telco-ran/           # Telco RAN (SNO/3-node DU sites)
    ├── configuration/
    │   ├── source-crs/              # Base RAN source CRs (by operator)
    │   ├── kube-compare-reference/  # Templated CRs for kube-compare
    │   ├── argocd/                  # ArgoCD deployment + examples
    │   └── extra-manifests-builder/ # MachineConfig builders
    └── install/
```

## Use Model Overview

| Use Model | Purpose | Topology | First Branch |
|-----------|---------|----------|--------------|
| **telco-core** | Large multi-node clusters running vDU/vCU workloads with performance tuning | 3+ control plane, N workers with MachineConfigPools | release-4.14 |
| **telco-hub** | Management/hub cluster running ACM, GitOps, TALM for fleet management | Standard HA cluster, partially disconnected | release-4.18 |
| **telco-ran** | Edge SNO or 3-node clusters for RAN DU workloads, managed via ZTP | SNO, 3-node compact, or standard | release-4.19 |

## Branch Availability Matrix

| Component | 4.14 | 4.15 | 4.16 | 4.17 | 4.18 | 4.19 | 4.20 | 4.21 | main |
|-----------|------|------|------|------|------|------|------|------|------|
| telco-core | 44 | 50 | 113 | 142 | 142 | 147 | 152 | 170 | 170 |
| telco-hub | — | — | — | — | 35 | 95 | 165 | 194 | 194 |
| telco-ran | — | — | — | — | — | 297 | 303 | 301 | 305 |

Values = number of YAML files per use model on that branch.

---

## Operator & Feature Inventory

### Operators by Use Model

| Operator / Feature | telco-core | telco-hub | telco-ran | Category |
|--------------------|:----------:|:---------:|:---------:|----------|
| SR-IOV Operator | required | — | required | Networking |
| NMState Operator | required | — | optional | Networking |
| MetalLB | required | — | — | Networking |
| Multi-network Policy | required | — | — | Networking |
| OVN-Kubernetes (Network.operator) | required | — | — | Networking |
| NROP (NUMA Resources) | required | — | — | Scheduling |
| Secondary Scheduler | required | — | — | Scheduling |
| Node Tuning (PerformanceProfile) | required | — | required | Performance |
| Tuned (TunedPerformancePatch) | required | — | required | Performance |
| ODF (external) | required | — | — | Storage |
| ODF (internal) | — | optional | — | Storage |
| LSO (Local Storage) | — | optional | optional | Storage |
| LVM Storage | — | — | optional | Storage |
| PTP Operator | — | — | required | Timing |
| SRIOV-FEC / Accelerators | — | — | optional | Acceleration |
| GitOps / ArgoCD | — | required | — | Management |
| ACM (Advanced Cluster Management) | — | required | — | Management |
| MCE (Multi Cluster Engine) | — | required | — | Management |
| TALM (Topology Aware LM) | upgrade | required | — | Lifecycle |
| LCA (Lifecycle Agent) | — | — | optional | Lifecycle |
| Cluster Logging (Vector) | optional | optional | required | Observability |
| Cert-Manager | optional | optional | — | Security |
| OADP (Data Protection) | — | optional | optional | Backup |

### Operator Introduction Timeline

| Release | New Operators / Features |
|---------|-------------------------|
| 4.14 | SR-IOV, NROP, Scheduler, PerformanceProfile, ODF-external, MetalLB (telco-core initial) |
| 4.15 | NMState, Multi-network Policy added to telco-core |
| 4.16 | kube-compare reference CRs, logging (CLO v5→v6 migration), SriovNetworkPoolConfig, ClusterVersion, kernel modules, monitoring-config |
| 4.17 | PolicyGenerator CRs (core-baseline/overlay/upgrade), template-values, install artifacts (SiteConfig), kdump, mount-namespace, upgrade-ack |
| 4.18 | **telco-hub introduced** (GitOps, LSO, ODF-internal, ACM, TALM, observability), ClusterLogging5Cleanup, ClusterLogOperatorStatus |
| 4.19 | **telco-ran introduced** (full RAN DU profile: PTP, LVM, SRIOV-FEC, LCA, IBU, OADP, source-crs, kube-compare-reference, ArgoCD examples), IDMS replaces ICSP, core-finish PolicyGenerator, hub overlays+kustomization |
| 4.20 | core-upgrade-finish, infrastructure platform CR, subscription-validator, hub kube-compare refs, RAN source-crs reorganization (deprecated dir), aarch64 PerformanceProfiles, GNRD interface rename |
| 4.21 | cert-manager (core+hub+RAN), TunedPerformancePatch in kube-compare, observabilityRoutePolicy, ZTP extra-manifests-policy, PTP TBC/TTSC configs, crio-wipe removal |
| main | PTP GNRD configs (PtpConfigGnrdBcNoHoldover, PtpConfigGnrdTGM) |

---

## CR Catalogue by Use Model

### telco-core Reference CRs (release-4.21 / main)

**Required:**

| Category | CRs | Purpose |
|----------|-----|---------|
| networking | Network.yaml | OVN-Kubernetes cluster network config |
| networking | NMState*.yaml (4 files) | Node network configuration operator |
| networking/sriov | SriovSubscription*.yaml, SriovOperatorConfig.yaml, sriovNetwork.yaml, sriovNetworkNodePolicy.yaml | SR-IOV network device plugin |
| networking/metallb | metallb*.yaml, bgp-*.yaml, bfd-profile.yaml, addr-pool.yaml, community.yaml, service.yaml | Load balancer for bare metal |
| networking/multinetworkpolicy | multiNetworkPolicy*.yaml | Network policies for secondary networks |
| performance | PerformanceProfile.yaml, TunedPerformancePatch.yaml | CPU isolation, hugepages, real-time kernel tuning |
| scheduling | NROP*.yaml, sched.yaml, Scheduler.yaml | NUMA-aware scheduling |
| storage/odf-external | odf*.yaml, ocs-external-storagecluster.yaml, rook-ceph secret | External Ceph storage |
| other | catalog-source.yaml, idms.yaml, operator-hub.yaml | Disconnected registry, operator catalogue |

**Optional:**

| Category | CRs | Purpose |
|----------|-----|---------|
| logging | ClusterLog*.yaml (8 files) | Cluster logging with Vector |
| cert-manager | certManager*.yaml, apiServer*.yaml, ingress*.yaml (8 files) | TLS certificate management |
| networking/sriov | SriovNetworkPoolConfig.yaml | SR-IOV pool configuration |
| networking/multus | mc_rootless_pods_selinux.yaml | SELinux for rootless TAP CNI pods |
| other | kdump-*.yaml, mount_namespace_config_*.yaml, sctp_module_mc.yaml, load-kernel-modules*.yaml, monitoring-config-cm.yaml, ClusterVersion.yaml, upgrade-ack.yaml, networkAttachmentDefinition.yaml, nodeNetworkConfigurationPolicy.yaml, server-cert-openshift-config-copy.yaml |

### telco-hub Reference CRs (release-4.21 / main)

**Required:**

| Category | CRs | Purpose |
|----------|-----|---------|
| acm | acm*.yaml, observability*.yaml, pullSecret*.yaml, thanosSecret*.yaml (22 files) | ACM hub, MCE, observability, policy distribution |
| gitops | gitops*.yaml, argocd-*.yaml, ztp-*.yaml, addPlugins*.yaml, clusterrole*.yaml (22 files) | GitOps operator, ArgoCD, ZTP plugin policies |
| registry | idms-*.yaml, itms-*.yaml, catalog-source.yaml, image-config.yaml, operator-hub.yaml, registry-ca.yaml | Disconnected mirror registry |
| talm | talmSubscription.yaml | Topology Aware Lifecycle Manager |

**Optional:**

| Category | CRs | Purpose |
|----------|-----|---------|
| odf-internal | odf*.yaml, storageCluster.yaml, odfReady.yaml | Internal ODF storage |
| lso | lso*.yaml, lsoLocalVolume.yaml | Local storage for ODF backing |
| logging | clusterLog*.yaml (7 files) | Cluster logging |
| cert-manager | certManager*.yaml, apiServer*.yaml, ingress*.yaml (12 files) | TLS management with ACM policy distribution |
| backup-recovery | objectBucketClaim.yaml, dataProtectionApplication.yaml, backupSchedule.yaml, restore.yaml, policy-backup.yaml | OADP backup/recovery |

### telco-ran Source CRs (release-4.21 / main)

Organized by operator subdirectory:

| Operator/Category | Key CRs | Purpose |
|-------------------|---------|---------|
| sriov-operator | SriovSubscription*.yaml, SriovOperatorConfig*.yaml, SriovNetwork.yaml, SriovNetworkNodePolicy*.yaml, SriovOperatorConfigForSNO.yaml | SR-IOV for DU workloads |
| ptp-operator | PtpSubscription*.yaml, PtpOperatorConfig*.yaml | PTP timing operator |
| ptp-operator/configuration | PtpConfig{Slave,Master,Boundary,GmWpc,DualCard,ThreeCard,ForHA,DualFollower,GnrdTGM,GnrdBcNoHoldover,TBCWpc,TTSCWpc,ThreeCardTBCWpc,DualCardTBCWpc}*.yaml (18+ configs) | PTP clock profiles (GM, BC, OC, T-BC, T-TSC) |
| node-tuning-operator | PerformanceProfile*.yaml (x86_64 + aarch64), TunedPerformancePatch.yaml, TunedPowerCustom.yaml | CPU isolation, hugepages, IRQ affinity |
| nmstate | NMState*.yaml | Node network configuration |
| machine-config | MachineConfigPool.yaml, RebootMachineConfig.yaml, MachineConfigSctp.yaml | MCP management, SCTP, reboot |
| cluster-logging | ClusterLog*.yaml (9 files) | Logging with Vector |
| cluster-tuning | ConsoleOperatorDisable.yaml, DisableOLMPprof.yaml, DisableSnoNetworkDiag.yaml, ReduceMonitoringFootprint.yaml, DefaultCatsrc.yaml, DisconnectedIDMS.yaml, OperatorHub.yaml | SNO footprint reduction |
| storage-lvm | StorageLVM*.yaml, LVMOperatorStatus.yaml | LVM storage for SNO |
| storage-lso | Storage*.yaml (7 files) | Local storage operator |
| lca | LcaSubscription*.yaml, LcaOperatorStatus.yaml | Lifecycle Agent for IBU |
| ibu | ImageBasedUpgrade*.yaml, PlatformBackupRestore*.yaml, ClusterVersion.yaml, ImageSignature.yaml | Image-based upgrade artifacts |
| data-protection | Oadp*.yaml (7 files) | Backup for IBU |
| image-registry | ImageRegistryConfig.yaml, ImageRegistryPV.yaml | Internal image registry for SNO |
| extra-manifest | MachineConfig CRs for kdump, SCTP, SRIOV kernel args, rcu-normal, container-mount-ns, disk-encryption, time-sync, GNRD interface rename, marketplace-ns | Day-0 install-time MachineConfigs |
| validatorCRs | informDuValidator*.yaml, rebootMachineConfigPoolValidator.yaml | DU profile validation |
| deprecated | MachineConfigContainerMountNS.yaml, StorageLocalVolume.yaml | Deprecated CRs (moved to subdirs) |

---

## Version Differences (Release-to-Release)

### release-4.14 → release-4.15

**Scope**: telco-core only (44 → 50 YAMLs)

| Change | Details |
|--------|---------|
| Added | NMState operator (4 CRs): NMState.yaml, NMStateNS.yaml, NMStateOperGroup.yaml, NMStateSubscription.yaml |
| Added | Multi-network policy (2 CRs): multiNetworkPolicyDenyAll.yaml, multiNetworkPolicyAllowPortProtocol.yaml |

### release-4.15 → release-4.16

**Scope**: telco-core (50 → 113 YAMLs) — major expansion

| Change | Details |
|--------|---------|
| Added | `reference-crs-kube-compare/` tree — templated CRs for cluster compliance checking |
| Added | Logging CRs: ClusterLogForwarder, ClusterLogServiceAccount + bindings |
| Added | SriovNetworkPoolConfig, SriovSubscriptionOperGroup, NROPSubscriptionOperGroup |
| Added | ClusterVersion.yaml, control-plane/worker-load-kernel-modules, monitoring-config-cm |
| Added | mc_rootless_pods_selinux (TAP CNI), operator-hub.yaml, Scheduler.yaml |
| Added | Tekton CI pipelines, build RPM lockfiles |
| Removed | Legacy logging CRs (ClusterLogForwarder, ClusterLogging from old location) |
| Breaking | CRs reorganized into `reference-crs/required/` and `reference-crs/optional/` hierarchy |

### release-4.16 → release-4.17

**Scope**: telco-core (113 → 142 YAMLs)

| Change | Details |
|--------|---------|
| Added | **PolicyGenerator CRs**: core-baseline.yaml, core-overlay.yaml, core-upgrade.yaml |
| Added | **Template values**: hw-types.yaml, regional.yaml ConfigMaps |
| Added | **Install artifacts**: example-standard.yaml (SiteConfig), extra-manifests/, custom-manifests/ |
| Added | kdump-master/worker, mount_namespace_config_master/worker, upgrade-ack |
| Added | ClusterLogForwarderDeleted (v5 cleanup), ClusterLogging.yaml |

### release-4.17 → release-4.18

**Scope**: telco-core (142, unchanged) + **telco-hub introduced** (35 YAMLs)

| Change | Details |
|--------|---------|
| Added | **telco-hub** use model: ACM (6 CRs), GitOps (16 CRs), TALM, registry, LSO, ODF-internal, logging |
| Added | telco-hub install artifacts: agent-config.yaml, install-config.yaml, imageset-config.yaml |
| Added | ClusterLogOperatorStatus, ClusterLogging5Cleanup (telco-core logging v5→v6 migration) |
| Removed | ClusterLogForwarderDeleted, ClusterLogging (replaced by v6 CRs) |

### release-4.18 → release-4.19

**Scope**: telco-core (142 → 147) + telco-hub (35 → 95) + **telco-ran introduced** (297 YAMLs)

| Change | Details |
|--------|---------|
| Added | **telco-ran** use model: complete RAN DU profile with source-crs, kube-compare-reference, ArgoCD examples, PolicyGenerator templates, siteconfig, extra-manifests |
| Added | telco-core: IDMS (replaces ICSP), core-finish.yaml, custom-manifests (mcp-worker-1/2/3) |
| Added | telco-hub: kustomization.yaml, overlay examples (ACM, GitOps, LSO, ODF, logging, registry), backup-recovery CRs, MCE, acmPerfSearch, acmMirrorRegistryCM, pull-secret-copy, thanosSecret |
| Removed | telco-core: ICSP (replaced by IDMS) |
| Removed | telco-hub: install-config/agent-config moved to openshift/ subdir |
| Breaking | ICSP → IDMS migration |

### release-4.19 → release-4.20

**Scope**: telco-core (147 → 152) + telco-hub (95 → 165) + telco-ran (297 → 303)

| Change | Details |
|--------|---------|
| Added | telco-core: core-upgrade-finish.yaml, infrastructure platform CR, subscription-validator, server-cert-openshift-config-copy |
| Added | **telco-hub kube-compare**: full reference-crs-kube-compare tree for all hub components |
| Added | telco-hub: pullSecret/thanosSecret Placement+PlacementBinding+MCSB CRs |
| Added | telco-ran: source-crs reorganization into operator subdirectories, deprecated/ dir, aarch64 PerformanceProfiles, GNRD interface rename, TunedPowerCustom |
| Removed | telco-ran: siteconfig/ moved, deprecated CRs to deprecated/, StorageLV from kube-compare |
| Breaking | telco-ran source-crs directory restructuring (flat → operator subdirs) |

### release-4.20 → release-4.21

**Scope**: telco-core (152 → 170) + telco-hub (165 → 194) + telco-ran (301)

| Change | Details |
|--------|---------|
| Added | **cert-manager** across all use models: telco-core (8 CRs + kube-compare), telco-hub (12 CRs + kube-compare + overlay), hub cert policy distribution |
| Added | telco-core: TunedPerformancePatch in kube-compare |
| Added | telco-hub: observabilityRoutePolicy, ztp-policies/extra-manifests-policy |
| Added | telco-ran: PTP T-BC/T-TSC configs (PtpConfigTBCWpc, PtpConfigTTSCWpc, DualCardTBCWpc, ThreeCardTBCWpc) |
| Removed | telco-ran: siteconfig/ ArgoCD examples (replaced by clusterinstance), crio-disable-wipe MachineConfigs, deprecated workload-partitioning |
| Breaking | SiteConfig → ClusterInstance migration for RAN provisioning |

### release-4.21 → main (next / 4.22)

| Change | Details |
|--------|---------|
| Added | PTP GNRD configs: PtpConfigGnrdBcNoHoldover.yaml, PtpConfigGnrdTGM.yaml |

---

## Configuration Architecture

### PolicyGenerator Flow (telco-core)

```
template-values/                PolicyGenerator YAML           reference-crs/
├── hw-types.yaml       ──►   ├── core-baseline.yaml    ──►  required/
├── regional.yaml              ├── core-overlay.yaml          optional/
└── (cluster-specific)         ├── core-upgrade.yaml
                               └── core-upgrade-finish.yaml
        │                              │
        ▼                              ▼
  ConfigMap values            ACM Policy objects
  (hw-type, zone,             (applied via TALM
   cluster-specific)           ClusterGroupUpgrade)
```

PolicyGenerator CRs reference `reference-crs/` and inject values from
`template-values/` ConfigMaps, producing ACM policies that TALM orchestrates.

### Upgrade Flow (telco-core)

1. **core-upgrade** PolicyGenerator creates upgrade policies:
   - `core-upgrade-prep-NN` — pre-upgrade: pause MCP, set upgrade-ack, channel
   - `core-upgrade-ocp-NN` — OCP version upgrade (ClusterVersion)
   - `core-upgrade-olm-NN` — OLM operator upgrades (Subscriptions)
   - `core-upgrade-validate-NN` — post-upgrade validation
2. **TALM ClusterGroupUpgrade** orchestrates the rollout across clusters
3. **core-upgrade-finish** — unpause MCP, release workers for drain/reboot

### ArgoCD Flow (telco-hub)

```
GitOps Operator ──► ArgoCD ──► kustomization.yaml
                                  ├── reference-crs/ (base)
                                  └── example-overlays-config/ (patches)
                                       ├── acm/
                                       ├── gitops/
                                       ├── lso/
                                       ├── odf/
                                       ├── registry/
                                       ├── logging/
                                       └── cert-manager/
```

Sync-wave ordering (-50 to 100) ensures dependencies are met:
- Wave -50: Namespaces
- Wave 0: Operators
- Wave 50: Operator configs
- Wave 100: Applications

### ZTP Flow (telco-ran)

```
Hub (ACM + GitOps)
  │
  ├── PolicyGenerator CRs ──► source-crs/ ──► ACM Policies
  │     (acmpolicygenerator/)    (base CRs)     (pushed to spoke)
  │
  ├── ClusterInstance CRs ──► Site provisioning
  │     (clusterinstance/)       (Agent-based installer)
  │
  └── extra-manifests ──► Day-0 MachineConfigs
        (extra-manifests-builder/)  (applied at install time)
```

### kube-compare Compliance

All three use models provide `reference-crs-kube-compare/` trees with:
- `metadata.yaml` — reference manifest list
- `ReferenceVersionCheck.yaml` — OCP version validation
- `required/` and `optional/` — templated CRs with Go template syntax

Run `kubectl cluster-compare -r <path>` to check cluster compliance.

---

## PTP Configuration Reference (telco-ran)

| Config | Role | Use Case |
|--------|------|----------|
| PtpConfigSlave | Ordinary Clock (OC) | Basic PTP follower |
| PtpConfigSlaveForEvent | OC + events | PTP follower with event notification |
| PtpConfigMaster | Grandmaster (GM) | GPS-synced time source |
| PtpConfigMasterForEvent | GM + events | GM with event notification |
| PtpConfigBoundary | Boundary Clock (BC) | Multi-port time relay |
| PtpConfigBoundaryForEvent | BC + events | BC with event notification |
| PtpConfigGmWpc | GM WPC | GM with Western Power Conversion GNSS |
| PtpConfigDualCardGmWpc | Dual-card GM WPC | Redundant GM with WPC |
| PtpConfigThreeCardGmWpc | Three-card GM WPC | Triple-redundant GM |
| PtpConfigForHA | HA | High-availability PTP |
| PtpConfigForHAForEvent | HA + events | HA with event notification |
| PtpConfigDualFollower | Dual follower | Two PTP sources |
| PtpConfigTBCWpc | T-BC WPC | Telecom Boundary Clock (4.21+) |
| PtpConfigDualCardTBCWpc | Dual T-BC WPC | Redundant T-BC (4.21+) |
| PtpConfigThreeCardTBCWpc | Three T-BC WPC | Triple T-BC (4.21+) |
| PtpConfigTTSCWpc | T-TSC WPC | Telecom Time Slave Clock (4.21+) |
| PtpConfigGnrdTGM | GNRD T-GM | Generic NIC Driver T-GM (main) |
| PtpConfigGnrdBcNoHoldover | GNRD BC | Generic NIC Driver BC without holdover (main) |

---

## PerformanceProfile Architecture Reference

| Field | Purpose | Typical Values |
|-------|---------|----------------|
| cpu.isolated | CPUs for workload (DU poll-mode) | `2-63` (SNO), `2-127` (standard) |
| cpu.reserved | CPUs for OS/platform | `0-1` |
| hugepages.defaultHugepagesSize | Default page size | `1G` |
| hugepages.pages | Allocation per NUMA node | `[{size: "1G", count: 32, node: 0}]` |
| realTimeKernel.enabled | RT kernel | `true` for DU |
| workloadHints.realTime | RT scheduling hints | `true` |
| workloadHints.highPowerConsumption | Disable power saving | `true` for max throughput |
| workloadHints.perPodPowerManagement | Per-pod C-states | `false` typically |
| numa.topologyPolicy | NUMA topology policy | `restricted` or `single-numa-node` |
| net.userLevelNetworking | Enable DPDK | `true` for DPDK workloads |
| additionalKernelArgs | Extra kernel params | `nohz_full`, `rcu_nocbs`, `isolcpus` |

Available for both **x86_64** and **aarch64** architectures (separate profiles since 4.20).

---

## Upgrade Path Analysis Reference

### Supported Upgrade Strategies

| Strategy | Mechanism | Use Model | Since |
|----------|-----------|-----------|-------|
| **Platform upgrade** (OCP) | ClusterVersion + TALM CGU | telco-core, telco-hub | 4.17 |
| **OLM operator upgrade** | Subscription channel/CSV + TALM CGU | telco-core, telco-hub | 4.17 |
| **Image-based upgrade (IBU)** | LCA + OADP backup + seed image | telco-ran (SNO) | 4.19 |
| **Manual upgrade** | Edit ClusterVersion directly | All (fallback) | 4.14 |

### Key Migration Considerations by Version Jump

| From → To | Critical Changes |
|-----------|-----------------|
| 4.14 → 4.15 | Add NMState operator CRs |
| 4.15 → 4.16 | Reorganize CRs into required/optional hierarchy; add kube-compare references |
| 4.16 → 4.17 | Adopt PolicyGenerator (core-baseline/overlay/upgrade); add install artifacts |
| 4.17 → 4.18 | Deploy telco-hub if managing fleet; CLO v5 → v6 migration |
| 4.18 → 4.19 | ICSP → IDMS migration; deploy telco-ran if using ZTP; add core-finish PolicyGenerator |
| 4.19 → 4.20 | telco-ran source-crs reorganization (flat → operator subdirs); add aarch64 profiles if needed; add core-upgrade-finish |
| 4.20 → 4.21 | Add cert-manager if needed; SiteConfig → ClusterInstance migration for RAN; remove crio-wipe MachineConfigs; add PTP T-BC/T-TSC configs |
| 4.21 → main | PTP GNRD configs only |

### Breaking Changes Summary

| Version | Breaking Change | Impact | Remediation |
|---------|----------------|--------|-------------|
| 4.16 | CR directory reorganization | PolicyGenerator paths change | Update PolicyGenerator CR file references |
| 4.18 | CLO v5 → v6 | ClusterLogging → ClusterLogForwarder API | Apply ClusterLogging5Cleanup, deploy new CLF CRs |
| 4.19 | ICSP → IDMS | ImageContentSourcePolicy deprecated | Replace ICSP with ImageDigestMirrorSet |
| 4.20 | telco-ran source-crs restructuring | Flat dir → operator subdirs | Update PolicyGenerator source-crs paths |
| 4.21 | SiteConfig → ClusterInstance | RAN provisioning API change | Migrate siteconfig/ to clusterinstance/ |
| 4.21 | crio-wipe removal | MachineConfig no longer needed | Remove 99-crio-disable-wipe MachineConfigs |

---

## Install Architecture

### telco-core Install

Uses **MCE SiteGen** utility:
- `example-standard-clusterinstance.yaml` — ClusterInstance CR for MCE
- `extra-manifests/` — Day-0 MachineConfigs (kdump, SCTP, mount-namespace, etc.)
- `custom-manifests/` — MachineConfigPool definitions (mcp-worker-1/2/3)

### telco-hub Install

Uses **Agent-based Installer (ABI)**:
- `install/openshift/install-config.yaml` — cluster config
- `install/openshift/agent-config.yaml` — agent boot config
- `install/mirror-registry/imageset-config.yaml` — mirror registry image set

### telco-ran Install

Uses **ZTP via ACM**:
- `argocd/example/clusterinstance/` — ClusterInstance CRs (SNO, 3-node, standard)
- `argocd/example/acmpolicygenerator/` — PolicyGenerator examples
- `extra-manifests-builder/` — Day-0 MachineConfig builders

---

## Common Patterns

### Subscription CR Pattern

All operator subscriptions follow:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: <operator>-subscription
  namespace: <operator-ns>
spec:
  channel: <channel>
  name: <package-name>
  source: <catalog-source>
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

### Namespace + OperatorGroup Pattern

Every operator requires:
1. `Namespace` — operator namespace
2. `OperatorGroup` — with `targetNamespaces` or cluster-scoped
3. `Subscription` — operator subscription

### kube-compare Template Pattern

Reference CRs in `reference-crs-kube-compare/` use Go templates:
```yaml
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  defaultNetwork:
    ovnKubernetesConfig:
      gatewayConfig:
        routingViaHost: {{ .routingViaHost | default true }}
```

### PolicyGenerator Merge Strategy

PolicyGenerator CRs use `complianceType`:
- `musthave` — fields in the policy must exist on the cluster
- `mustonlyhave` — cluster object must match exactly
- `inform` — report compliance without enforcing

---

## See Also

- [openshift-kni/telco-reference](https://github.com/openshift-kni/telco-reference) — source repository
- [openshift-kni/cnf-features-deploy](https://github.com/openshift-kni/cnf-features-deploy) — SiteGen utility
- [stolostron/policy-generator-plugin](https://github.com/stolostron/policy-generator-plugin) — PolicyGenerator plugin
- [openshift/kube-compare](https://github.com/openshift/kube-compare) — cluster compliance checker
- [open-cluster-management.io](https://open-cluster-management.io/) — ACM documentation
