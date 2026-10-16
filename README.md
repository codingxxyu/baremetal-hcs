# HCS Bare Metal ACP 4.4.1

固定环境：

| 集群 | 角色 | 架构 | 数量 |
|---|---|---|---|
| Global | 控制面 | amd64 | 3 |
| Workload | 控制面 | amd64 | 3 |
| Workload | Worker | arm64 | 1 |

不部署 Global Worker。安装设备一律 `/dev/elemental-install-target`。KCP / MachineDeployment 的 `maxSurge` 一律 `0`。

先设变量：

```bash
export PACK_DIR=/path/to/baremetal-hcs
export BOOTSTRAP_KUBECONFIG=/path/to/bootstrap-kubeconfig
export GLOBAL_KUBECONFIG=/path/to/global-kubeconfig
```

| 阶段 | 在哪执行 | kubeconfig |
|---|---|---|
| 1–6 | CAN / Bootstrap 节点 | `$BOOTSTRAP_KUBECONFIG`，集群名 `minialauda` |
| 物理机装 OS | 对应物理机 BMC / Live ISO | 不用 kubeconfig |
| 7–8 | CAN / 运维机 | `$GLOBAL_KUBECONFIG` |
| 9 运维 | 看对象时用对应 kubeconfig；看节点日志时登录那台机器 | 见各命令 |

---

## 1. 填值

对照文件：

```text
config/config.example.yaml
```

复制一份本地用：

```bash
cp "$PACK_DIR/config/config.example.yaml" "$PACK_DIR/config/config.local.yaml"
```

把 `config.local.yaml` 里的值手工填进编号 YAML / JSON。改对照文件不会自动改 manifests。

固定值：

```text
Kubernetes   v1.35.6-acp.9
CoreDNS      1.14.7-v4.4.7
etcd         v3.6.13-acp.2
containerd   2.2.8-3
pause        3.10
Kube-OVN     v4.4.12
OS           alaudaos.k.25
rootfs 仓    tkestack/kubeos-rootfs
ISO 仓       tkestack/kubeos-iso
平台账号     admin@cpaas.io
仓库账号     Registry
```

Catalog key：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25
v1.35.6-acp.9_arm64_alaudaos.k.25
```

`baremetal` 只是 archive 导入后的本地 source tag。YAML 里写的是 archive 对应的最终 tag：`<K25_AMD64_IMAGE_TAG>` / `<K25_ARM64_IMAGE_TAG>`。

每类文件改什么：

| 文件 | 改什么 |
|---|---|
| `global/bootstrap/02-kubeadm-provider-apprelease.yaml` | KIND 仓库、Kubeadm chart 版本 |
| `global/bootstrap/03-baremetal-provider-apprelease.yaml` | KIND 仓库、Bare Metal chart 版本、amd64 rootfs tag |
| `global/control-plane/01-registration-seedimage-amd64.yaml` | Bootstrap rootfs/ISO、amd64 tag |
| `global/control-plane/02-control-plane-pool.yaml` | 三个 Global inventory、网卡 |
| `global/control-plane/03-baremetal-cluster.yaml` | Global VIP、网卡、Bootstrap 仓库 |
| `global/control-plane/05-cluster.yaml` | Global CIDR、Bootstrap 仓库 |
| `global/control-plane/06-kubeadm-control-plane.yaml` | SSH key |
| `global/bootstrap/04-dcs-import-extra-resources.yaml` | 三个实际 Global inventory |
| `global/bootstrap/05-installer-config.example.json` | 平台 host、仓库、密码、VIP、CIDR |
| `global/control-plane/07-finished-kubeadm-provider-apprelease.yaml` | 平台仓库、Kubeadm chart 版本 |
| `global/control-plane/08-finished-baremetal-provider-apprelease.yaml` | 平台仓库、host、amd64/arm64 catalog |
| `workload/control-plane/01`–`06` | 平台仓库、Workload inventory/VIP/CIDR、SSH key |
| `workload/worker/01`–`05` | ARM tag、`linux/arm64`、ARM inventory、SSH key |

kubelet patch 里不要写 `staticPodPath`，包括 `null`。

---

## 2. 推 KubeOS 镜像

节点：CAN / Bootstrap。

先按现场策略配好仓库证书信任，不要关 TLS 校验。

```bash
export OS_ARCHIVE=<ACP_4_4_1_K25_AMD64_ARCHIVE>
export OS_TAG=<K25_AMD64_IMAGE_TAG>
export REGISTRY=<PLATFORM_REGISTRY>:11443

ctr -n bare-metal-os images rm \
  kubeos-rootfs:baremetal kubeos-iso:baremetal 2>/dev/null || true
ctr -n bare-metal-os images import --all-platforms --no-unpack "$OS_ARCHIVE"
ctr -n bare-metal-os images ls -q | grep -E 'kubeos-(rootfs|iso):baremetal$'

for repo in kubeos-rootfs kubeos-iso; do
  ctr -n bare-metal-os images push \
    -u 'Registry:<REGISTRY_PASSWORD>' \
    "$REGISTRY/tkestack/$repo:$OS_TAG" \
    "$repo:baremetal"
done
```

arm64 再跑一遍，换：

```text
OS_ARCHIVE=<ACP_4_4_1_K25_ARM64_ARCHIVE>
OS_TAG=<K25_ARM64_IMAGE_TAG>
```

两个架构的最终 tag 不能互相覆盖。Workload 创建前，平台仓库里要同时有 amd64 / arm64 的 `kubeos-rootfs` 和 `kubeos-iso`。

---

## 3. Bootstrap：装 provider，出 Global ISO

节点：CAN / Bootstrap。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

```bash
export KUBECONFIG="$BOOTSTRAP_KUBECONFIG"
kubectl get nodes
kubectl get storageclass
kubectl -n cpaas-system get pods
```

确认 KIND 名是 `minialauda`，有默认 StorageClass。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/01-dex-tls-bootstrap.yaml"
kubectl apply -f "$PACK_DIR/global/bootstrap/02-kubeadm-provider-apprelease.yaml"
kubectl apply -f "$PACK_DIR/global/bootstrap/03-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
```

`03` 的 catalog 只保留 amd64。Ready 后再继续。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/01-registration-seedimage-amd64.yaml"
kubectl -n cpaas-system get seedimage global-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

`status.downloadURL` 为空不要下 ISO。这个 ISO 只给三台 Global 控制面用。

---

## 4. 三台 Global 物理机装系统

节点：每台 Global 物理机的 BMC / Live ISO。不用 kubectl。

```bash
lsblk -d -e7 -o NAME,PATH,SIZE,MODEL,SERIAL,WWN,HCTL
ln -sfn /dev/disk/by-id/wwn-<这台机器的 OS 盘 WWN> \
  /dev/elemental-install-target
readlink -f /dev/elemental-install-target
test -b "$(readlink -f /dev/elemental-install-target)"
```

WWN 对照 `config/config.local.yaml` 的 `hosts`。不要链到 ISO、loop 或数据盘。

装完：弹出虚拟光驱，BMC 改成硬盘优先，等机器回到网络。

回到 CAN 节点：

```bash
export KUBECONFIG="$BOOTSTRAP_KUBECONFIG"
kubectl -n cpaas-system get machineinventory -o wide
```

三个 inventory Ready 后：

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/02-control-plane-pool.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/03-baremetal-cluster.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/04-control-plane-machine-template.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/05-cluster.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/06-kubeadm-control-plane.yaml"
```

`02` 三个 inventory 名称必须互不相同。`06` 是 `replicas: 3`、`maxSurge: 0`。

```bash
kubectl -n cpaas-system get cluster global
kubectl -n cpaas-system get kubeadmcontrolplane global-kcp
kubectl -n cpaas-system get machine,baremetalmachine -o wide
```

三台 Global 控制面 Ready 后再做 handoff。

---

## 5. Handoff：installer

节点：CAN / Bootstrap。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

先读实际 inventory：

```bash
kubectl -n cpaas-system get baremetalmachine \
  -l cluster.x-k8s.io/cluster-name=global \
  -o custom-columns='NAME:.metadata.name,INVENTORY:.status.machineInventoryRef.name'
```

填进 `global/bootstrap/04-dcs-import-extra-resources.yaml`，然后：

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/04-dcs-import-extra-resources.yaml"
```

这个文件不导入 SeedImage、token、plan、bootstrap TLS。

installer JSON 复制到本机，不要改仓库里的 example：

```bash
cp "$PACK_DIR/global/bootstrap/05-installer-config.example.json" \
  /root/installer-config.json
```

`/root/installer-config.json` 必填：

| 字段 | 值 |
|---|---|
| `registry.domain` | `<PLATFORM_REGISTRY>:11443` |
| `registry.username` | `Registry` |
| `registry.password` | registry 明文密码的 base64 |
| `basic.username` | `admin@cpaas.io` |
| `console.host` / `globalHost` | `<PLATFORM_HOST>` |
| `cluster.features.ha.vip` | `<GLOBAL_CONTROL_PLANE_VIP>` |
| `cluster.clusterCIDR` / `serviceCIDR` | 与 Global Cluster YAML 相同 |
| `deployMode` | `normal` |
| `hostIP` | `<BOOTSTRAP_HOST_IP>` |

`features.ha` 是 installer 收 API VIP 的字段。

```bash
kubectl -n cpaas-system get pods -l service_name=cpaas-installer -o wide
export INSTALLER_IP=$(kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o jsonpath='{.items[0].status.podIP}')
test -n "$INSTALLER_IP"
kubectl -n cpaas-system logs -f \
  -l service_name=cpaas-installer --all-containers=true --prefix=true
```

另开一个终端提交。必须打 installer Pod IP 的 `8080`，不要打平台 VIP：

```bash
curl -fS -X POST \
  "http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs" \
  -H 'Content-Type: application/json' \
  --data-binary @/root/installer-config.json
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
```

---

## 6. Finished Global provider 和 catalog CM

节点：CAN / 运维机。kubeconfig：`$GLOBAL_KUBECONFIG`。

installer 成功后拿到 `global-kubeconfig`：

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl get nodes
kubectl -n cpaas-system get cluster global \
  -o jsonpath='{.metadata.annotations.cpaas\.io/registry-address}'; echo
kubectl -n cpaas-system get baremetalcluster global \
  -o jsonpath='{.metadata.annotations.cpaas\.io/registry-address}'; echo
```

`BaremetalCluster/global` 如果还是 Bootstrap 仓库，只 patch 这个对象到平台仓库。不要重新 apply `global/control-plane/03` 或 `05`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/07-finished-kubeadm-provider-apprelease.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/08-finished-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
```

`08` 必须同时有 amd64 和 arm64 catalog。

真正给 Workload 用的是 ConfigMap，不是 AppRelease 里那一段：

```bash
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

`.data` 至少要有：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25=<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
v1.35.6-acp.9_arm64_alaudaos.k.25=<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>
```

缺 key 就 merge patch，不要 replace 整个 ConfigMap：

```bash
kubectl -n cpaas-system patch configmap elemental-image-catalog \
  --type merge \
  -p "{\"data\":{\"v1.35.6-acp.9_amd64_alaudaos.k.25\":\"<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>\",\"v1.35.6-acp.9_arm64_alaudaos.k.25\":\"<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>\"}}"
kubectl -n cpaas-system get configmap elemental-image-catalog \
  -o jsonpath='{.data}'; echo
```

ConfigMap 对象不存在时，先查 `08` 的 AppRelease 是否 Ready。两个 key 都对了，再创建 Workload。

---

## 7. Workload 控制面

节点：CAN / 运维机。kubeconfig：`$GLOBAL_KUBECONFIG`。

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl apply -f "$PACK_DIR/workload/control-plane/01-registration-seedimage-amd64.yaml"
kubectl -n cpaas-system get seedimage hcs-workloadcluster-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

这个 ISO 只给三台 Workload amd64 控制面。不要复用 Global ISO。

每台 Workload 控制面按第 4 章做 WWN symlink 后安装。装完回到 CAN：

```bash
kubectl -n cpaas-system get machineinventory \
  -o custom-columns='NAME:.metadata.name,READY:.status.ready'
```

三个 inventory 名称必须不同。填进 `workload/control-plane/02-control-plane-pool.yaml` 后再 apply。如果报 `Duplicate value`，改第三个 name，不要删 Cluster。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/02-control-plane-pool.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/03-baremetal-cluster.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/04-control-plane-machine-template.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/05-cluster.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/06-kubeadm-control-plane.yaml"
kubectl get cluster hcs-workloadcluster
kubectl get kubeadmcontrolplane hcs-workloadcluster-control-plane
kubectl get machines -l cluster.x-k8s.io/cluster-name=hcs-workloadcluster -o wide
```

`06` 里不要写 `staticPodPath`。

---

## 8. Workload ARM Worker

节点：CAN / 运维机 apply；ARM 物理机装系统。kubeconfig：`$GLOBAL_KUBECONFIG`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/01-registration-seedimage-arm64.yaml"
kubectl -n cpaas-system get seedimage \
  hcs-workloadcluster-arm64-worker-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

确认：

```text
system-uri / baseImage = <K25_ARM64_IMAGE_TAG>
targetPlatform         = linux/arm64
device                 = /dev/elemental-install-target
```

ARM ISO 只给那一台 ARM 机器。不要用 amd64 ISO。

ARM inventory Ready 后：

```bash
kubectl apply -f "$PACK_DIR/workload/worker/02-worker-pool-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/03-worker-machine-template-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/04-worker-kubeadm-config-template-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/05-worker-machine-deployment-arm64.yaml"
kubectl get machines -l cluster.x-k8s.io/cluster-name=hcs-workloadcluster -o wide
```

`04` 必须有：

```text
volume-plugin-dir: /opt/libexec/kubernetes/kubelet-plugins/volume/exec/
```

`05` 是 `replicas: 1`、`maxSurge: 0`、`maxUnavailable: 1`。

---

## 9. 运维

先看日志，再删单个对象。删 `Cluster` 是最后手段，且只能删 Workload，不能删 `Cluster/global`。

### 9.1 时间同步

所有 Global 控制面、Workload 控制面、ARM Worker 都要先对时：

```bash
date -u
timedatectl show -p NTPSynchronized --value
chronyc tracking
```

漏做了对时：先把相关物理机时间同步好，再删卡住的 `Machine`，看控制器会不会重建。不要一上来删 Cluster。

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system get machine -o wide
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system delete machine <FAILED_MACHINE_NAME>
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system get machine,baremetalmachine -o wide \
  -l cluster.x-k8s.io/cluster-name=<CLUSTER_NAME>
```

### 9.2 CAN 上看 installer

kubeconfig 仍是 `$BOOTSTRAP_KUBECONFIG`：

```bash
kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cpaas-installer \
  --all-containers=true --prefix=true
kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" \
  -n cpaas-system logs -l service_name=cpaas-installer \
  --previous --all-containers=true
```

进度：

```bash
export INSTALLER_IP=$(kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" \
  -n cpaas-system get pods -l service_name=cpaas-installer \
  -o jsonpath='{.items[0].status.podIP}')
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
```

### 9.3 触发重新安装

资源已经 apply 完但卡住时，先删失败的 `Machine`：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system get machine,baremetalmachine -o wide
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system delete machine <FAILED_MACHINE_NAME>
```

旧对象被清掉、新对象被拉起，说明上一轮已经清完并开始重装。

只有单个对象删了也拉不起来、并且现场允许重建这个 Workload 时，才删 Workload Cluster：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system delete cluster hcs-workloadcluster
```

等级联对象清完，再按顺序 apply：

```text
workload/control-plane/03-baremetal-cluster.yaml
workload/control-plane/04-control-plane-machine-template.yaml
workload/control-plane/05-cluster.yaml
workload/control-plane/06-kubeadm-control-plane.yaml
workload/worker/03-worker-machine-template-arm64.yaml
workload/worker/04-worker-kubeadm-config-template-arm64.yaml
workload/worker/05-worker-machine-deployment-arm64.yaml
```

pool 也被清掉时，先确认 inventory Ready，再 apply `02`。已经存在的 Registration / SeedImage / inventory 不要重复创建。

`ImageCatalogMiss`：先补 catalog CM，再只删失败的 `BaremetalMachine`。

### 9.4 登录机器看 cloud-init / elemental-agent

```bash
sudo less /var/log/cloud-init-output.log
sudo journalctl -fu elemental-system-agent
```

unit 名如果是 `elemental-agent-system`：

```bash
sudo journalctl -fu elemental-agent-system
```

不确定就先查：

```bash
systemctl list-units --type=service --all | grep -Ei 'elemental|cloud-init'
```

Live ISO 还不能 SSH 时，在 BMC 上看：

```bash
journalctl -f | grep -Ei 'elemental|register|install|cloud-init' || true
```

### 9.5 kubelet / containerd

```bash
sudo journalctl -fu kubelet
sudo journalctl -fu containerd
sudo crictl ps -a
sudo systemctl status kubelet containerd
```

节点上如果已经出现 `"staticPodPath": null`，先改对应 YAML，再按 9.3 重建那台 Machine。

### 9.6 CAN 上看 cluster-transformer

Bootstrap：

```bash
kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cluster-transformer \
  --all-containers=true --prefix=true
```

Finished Global：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cluster-transformer \
  --all-containers=true --prefix=true
```

installer 看 Global 首次安装；transformer 看后续插件和集群变更。

---

不要提交 kubeconfig、密码、ISO、archive、Registration URL、私钥。不要关 TLS 校验。不要删 `Cluster/global`。
