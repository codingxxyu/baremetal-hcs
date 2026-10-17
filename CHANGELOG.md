# Changelog

## 0.1.6-draft — 2026-10-17

Workload cluster name is optional, not a fixed `hcs-workloadcluster`.

- YAML under `workload/` uses `<WORKLOAD_CLUSTER_NAME>` as the Cluster / BaremetalCluster / KCP / pool / Registration / SeedImage / MachineDeployment name prefix.
- `config.example.yaml` keeps `hcs-workloadcluster` as the example value. The next site may choose another name as long as it is not `global` and the same name is used on every Workload object.
- README chapter 0 and the Workload apply commands follow the placeholder.

## 0.1.5-draft — 2026-10-17

Field-handbook rewrite and YAML structural audit. Manifest objects are unchanged except comments.

- README: one shared handoff path (chapters 6–7) for both host scenes. Physical scene no longer duplicates installer / platform-registry steps.
- README: physical Global Pool–KCP and Workload Pool–KCP / Worker MD now point at the HCS sections that already document the same files, placeholders, and apply commands.
- README: operations move from chapter 15 to chapter 13. Wording is tightened; gates, commands, and placeholder tables stay.
- YAML audit: all YAML/JSON parse; embedded kubelet JSON is valid. HCS Registration keeps `/dev/vda`; ISO twins keep `/dev/elemental-install-target`. Pools omit `networkDevice`; both BaremetalCluster objects keep it. Duplicate MachineRegistration names remain intentional twins.
- KCP / ARM worker comments record `NeverVerify`, omitted `staticPodPath`, and K25 volume-plugin dirs.
- `config.example.yaml` pack version `0.1.5-draft`; package filename tokens (`<ACP_4_4_1_HYBRID_CORE_PACKAGE>`, `<KUBEADM_PROVIDER_ALL_PACKAGE>`, `<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>`) are named next to the worksheet fields.

## 0.1.4-draft — 2026-10-17

Splits the handbook into two field modules. YAML objects are unchanged.

- Scene 1 (chapters 5–9): HCS virtual-machine Bare Metal. Registration-only files, qcow2, cloud-init. No SeedImage.
- Scene 2 (chapters 10–14): physical Bare Metal. Registration+SeedImage, Live ISO, install-target symlink. No qcow2.
- Chapters 1–4 stay shared. Operations move to chapter 15.
- Removes 路径 A / 路径 B wording so a site opens only the matching module.

## 0.1.3-draft — 2026-10-17

Field-handbook wording pass. YAML objects are unchanged except comments.

- Replaces leftover CAN / Bootstrap / Finished Global / 客机 / 运维机 / 操作机 with 引导节点、引导集群、成品 Global、虚机.
- Adds a name table at the top of the README.
- Fixes kubeconfig chapter mapping: extract `global-kubeconfig` in 5.8, not chapter 6.
- NTP now covers HCS VMs as well as physical hosts.
- Notes that the cloud-init YAML check needs PyYAML.
- Explains the installer `dcs-` ConfigMap name and that installer does not rewrite BaremetalCluster.

## 0.1.2-draft — 2026-10-17

Adds the HCS qcow2 host-join path next to the existing ISO path.

- Keeps every ISO `01-registration-seedimage-*.yaml` file unchanged.
- Adds architecture-named Registration-only files for VMs that cannot mount ISO: `global/control-plane/01-registration-amd64.yaml`, `workload/control-plane/01-registration-amd64.yaml`, `workload/worker/01-registration-arm64.yaml`.
- Adds `config/cloud-init-user-data.example.yaml` for HCS 用户数据注入.
- Documents two host-join methods in the README; Pool / Cluster / KCP stay the same after Inventory is Ready.
- Records the two KubeOS qcow2 templates (UEFI, `/dev/vda`) and the optional vda4 expand commands.

## 0.1.1-draft — 2026-10-17

Field-delivery rewrite of the README. YAML-only pack is unchanged.

- Starts from Core Package `setup.sh` (KIND `minialauda`, embedded registry, cpaas-installer) instead of assuming a bootstrap kubeconfig already exists.
- Documents the four registry addresses: `127.0.0.1:11443`, `<KIND_REGISTRY_ADDRESS>:11443`, `<BOOTSTRAP_HOST_IP>:11443`, `<PLATFORM_REGISTRY>:11443`.
- Pushes amd64 OS and both provider charts to the bootstrap registry before Global ISO; pushes amd64+arm64 OS and charts to the platform registry only after installer Success.
- Adds site-entry checks, NTP before install, CIDR non-overlap, SSH key, and installer password base64 encoding.
- Adds a gate table, SeedImage `downloadURL` wait, `global-kubeconfig` extraction from Secret, installer JSON validation, handoff three-item check, and the exact BaremetalCluster annotate command.
- Moves time sync to pre-install; keeps chapter 10 for missed-NTP recovery and other operations.

## 0.1.0-draft — 2026-10-16

First HCS Bare Metal delivery pack for the fixed ACP 4.4.1 single-site deployment.

- Uses one authoritative `config/config.example.yaml` reference file.
- Removes duplicate release/site worksheets and the unused root package metadata file.
- Keeps only the required Global bootstrap/control-plane and Workload control-plane/worker flows.
- Fixes the topology to three Global amd64 control planes, three Workload amd64 control planes, and one Workload arm64 worker.
- Removes the unused Global Worker extension and unsupported load-balancer alternatives.
- Replaces hardcoded site CIDRs, NIC values, inventory names, and SSH values with consistent placeholders.
- Uses KubeOS `kubeos-rootfs` and `kubeos-iso` with separate amd64/arm64 catalog entries.
- Omits `staticPodPath` from every kubelet patch and keeps the K25 worker `volume-plugin-dir` setting.
- Keeps `/dev/elemental-install-target` as the shared installation device.
- Documents installer handoff, registry transitions, SeedImage gates, disk safety, and practical troubleshooting.
- Adds a final operations chapter for installer logs, rebuild triggers, cloud-init, elemental-agent, kubelet/containerd, and cluster-transformer.
- Rewrites the README as a delivery handbook: Global then Workload, each step naming the YAML and the placeholders to replace.
- Corrects README kubeconfig stage mapping and removes leftover Bootstrap placeholders from Workload comments.
- No scripts, renderer, commit, or push are included in this draft.
