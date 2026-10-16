# Changelog

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
- No scripts, renderer, commit, or push are included in this draft.
