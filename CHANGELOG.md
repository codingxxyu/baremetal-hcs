# Changelog

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
