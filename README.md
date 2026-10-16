# HCS Bare Metal ACP 4.4.1 交付手册

## 0. 交付范围

| 集群 | 角色 | 架构 | 数量 | 目录 |
|---|---|---|---|---|
| Global | 控制面 | amd64 | 3 | `global/` |
| Workload | 控制面 | amd64 | 3 | `workload/control-plane/` |
| Workload | Worker | arm64 | 1 | `workload/worker/` |

不部署 Global Worker。安装设备一律 `/dev/elemental-install-target`。控制面和 Worker 的 `maxSurge` 一律 `0`。

固定版本：

```text
Kubernetes   v1.35.6-acp.9
CoreDNS      1.14.7-v4.4.7
etcd         v3.6.13-acp.2
containerd   2.2.8-3
pause        3.10
Kube-OVN     v4.4.12
OS           alaudaos.k.25
rootfs       tkestack/kubeos-rootfs
ISO          tkestack/kubeos-iso
平台账号     admin@cpaas.io
仓库账号     Registry
```

Catalog key：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25
v1.35.6-acp.9_arm64_alaudaos.k.25
```

`baremetal` 只是 archive 导入后的本地 source tag。YAML 里必须写成实际最终 tag：`<K25_AMD64_IMAGE_TAG>`、`<K25_ARM64_IMAGE_TAG>`。

kubelet patch 不要写 `staticPodPath`，包括 `null`。

### 0.1 执行位置和 kubeconfig

先设变量：

```bash
export PACK_DIR=/path/to/baremetal-hcs
export BOOTSTRAP_KUBECONFIG=/path/to/bootstrap-kubeconfig
export GLOBAL_KUBECONFIG=/path/to/global-kubeconfig
```

| 阶段 | 执行节点 | kubeconfig | 目标集群 |
|---|---|---|---|
| 第 2–5 章 | CAN / Bootstrap 节点 | `$BOOTSTRAP_KUBECONFIG` | KIND `minialauda` |
| 物理机装 OS | 对应物理机 BMC / Live ISO | 无 | 该物理机 |
| 第 6 章起 | CAN / 运维机 | `$GLOBAL_KUBECONFIG` | Finished Global |
| 第 9 章 | 见各命令 | 见各命令 | 见各命令 |

### 0.2 对照文件

唯一对照文件：

```text
config/config.example.yaml
```

本地复制：

```bash
cp "$PACK_DIR/config/config.example.yaml" "$PACK_DIR/config/config.local.yaml"
```

后续每一步列出的变量，都从 `config.local.yaml` 取实际值，手工替换到该步 YAML / JSON。改对照文件不会自动改 manifests。

密码、kubeconfig、ISO、archive、Registration URL、私钥不要提交。

---

## 1. 开始前必须填好的值

先把 `config.local.yaml` 填完，再改 YAML。现场核对盘符 WWN 时用 `hosts`，这些 WWN 不写入共享 YAML。

| 变量 | 用在哪 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | Bootstrap TLS、Bootstrap provider、Global Registration rootfs、Global 首次 Cluster/BaremetalCluster、Global KCP etcd SAN、installer `hostIP` |
| `<KIND_REGISTRY_ADDRESS>` | Bootstrap AppRelease `repoURL` / registry、Global SeedImage `baseImage` |
| `<PLATFORM_HOST>` | Finished provider、installer console |
| `<PLATFORM_REGISTRY>` | 推镜像、installer registry、Finished provider、Workload 全部 registry |
| `<GLOBAL_CONTROL_PLANE_VIP>` | Bootstrap Bare Metal provider、Global BaremetalCluster、Global KCP etcd SAN、installer `features.ha.vip` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | Workload BaremetalCluster、Workload KCP etcd SAN |
| `<NETWORK_DEVICE>` | 所有 inventory pool、两个 BaremetalCluster |
| `<GLOBAL_POD_CIDR>` `<GLOBAL_SERVICE_CIDR>` `<GLOBAL_KUBE_OVN_JOIN_CIDR>` | Global Cluster、installer JSON |
| `<WORKLOAD_POD_CIDR>` `<WORKLOAD_SERVICE_CIDR>` `<WORKLOAD_KUBE_OVN_JOIN_CIDR>` | Workload Cluster |
| `<SSH_AUTHORIZED_KEY>` | Global KCP、Workload KCP、ARM Worker KubeadmConfigTemplate |
| `<KUBEADM_PROVIDER_CHART_VERSION>` | Bootstrap / Finished Kubeadm AppRelease |
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | Bootstrap / Finished Bare Metal AppRelease |
| `<K25_AMD64_IMAGE_TAG>` | amd64 catalog、amd64 Registration/SeedImage |
| `<K25_ARM64_IMAGE_TAG>` | arm64 catalog、arm64 Registration/SeedImage |
| `<GLOBAL_CP_INVENTORY_1/2/3>` | Global pool、DCS import。装完机后再填实际名称 |
| `<WORKLOAD_CP_INVENTORY_1/2/3>` | Workload 控制面 pool。装完机后再填实际名称 |
| `<WORKLOAD_ARM64_WORKER_INVENTORY>` | ARM Worker pool。装完机后再填实际名称 |
| `Registry` / `<BASE64_REGISTRY_PASSWORD>` | 推镜像、installer JSON |
| `admin@cpaas.io` / `<BASE64_PLATFORM_ADMIN_PASSWORD>` | installer JSON 的平台账号，不是仓库账号 |

---

## 2. 推 KubeOS 镜像

**节点：** CAN / Bootstrap。

按现场策略配置仓库证书信任，不要关 TLS 校验。

先推 amd64：

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

再推 arm64，换：

```text
OS_ARCHIVE=<ACP_4_4_1_K25_ARM64_ARCHIVE>
OS_TAG=<K25_ARM64_IMAGE_TAG>
```

两个最终 tag 不能互相覆盖。Workload 创建前，平台仓库必须同时有：

```text
tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
tkestack/kubeos-iso:<K25_AMD64_IMAGE_TAG>
tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>
tkestack/kubeos-iso:<K25_ARM64_IMAGE_TAG>
```

---

# 第一部分：创建 Global 集群

本章全部在 CAN / Bootstrap 节点执行，kubeconfig 指向 KIND `minialauda`：

```bash
export KUBECONFIG="$BOOTSTRAP_KUBECONFIG"
```

---

## 3. Global：装 Bootstrap provider

先确认 Bootstrap 集群可用：

```bash
kubectl get nodes
kubectl get storageclass
kubectl -n cpaas-system get pods
```

KIND 名必须是 `minialauda`，并且有默认 StorageClass（SeedImage 需要 `20Gi` PVC）。

### 3.1 `global/bootstrap/01-dex-tls-bootstrap.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | Certificate `spec.commonName`、`spec.ipAddresses` |

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/01-dex-tls-bootstrap.yaml"
```

### 3.2 `global/bootstrap/02-kubeadm-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<KUBEADM_PROVIDER_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<KIND_REGISTRY_ADDRESS>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<KIND_REGISTRY_ADDRESS>:11443` |
| `<BOOTSTRAP_HOST_IP>` | `values.global.host`、`values.global.platformUrl`（`https://<BOOTSTRAP_HOST_IP>`） |

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/02-kubeadm-provider-apprelease.yaml"
```

### 3.3 `global/bootstrap/03-baremetal-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<KIND_REGISTRY_ADDRESS>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<KIND_REGISTRY_ADDRESS>:11443` |
| `<BOOTSTRAP_HOST_IP>` | `values.global.host`、`values.global.platformUrl`（`https://<BOOTSTRAP_HOST_IP>`）、`values.elemental.server.url`（`https://<BOOTSTRAP_HOST_IP>:12443`） |
| `<K25_AMD64_IMAGE_TAG>` | `values.provider.imageCatalog.images.v1.35.6-acp.9_amd64_alaudaos.k.25.tag` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `values.provider.handoffHook.controlPlaneVIP` |

这一步 catalog **只填 amd64**。不要在 Bootstrap 阶段加 arm64 catalog。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/03-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
```

两个 AppRelease Ready 后再继续。

---

## 4. Global：出 ISO，装三台控制面，创建 Cluster

### 4.1 `global/control-plane/01-registration-seedimage-amd64.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | MachineRegistration `system-uri`，写成 `<BOOTSTRAP_HOST_IP>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>` |
| `<KIND_REGISTRY_ADDRESS>` | SeedImage `baseImage`，写成 `<KIND_REGISTRY_ADDRESS>:11443/tkestack/kubeos-iso:<K25_AMD64_IMAGE_TAG>` |
| `<K25_AMD64_IMAGE_TAG>` | `system-uri` 和 `baseImage` 的 tag |

固定值不要改：

```text
device          /dev/elemental-install-target
targetPlatform  linux/amd64
os id           alaudaos.k.25
```

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/01-registration-seedimage-amd64.yaml"
kubectl -n cpaas-system get seedimage global-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

`status.downloadURL` 为空不要下 ISO。这个 ISO 只给三台 Global 控制面，不要给 Workload 用。

### 4.2 三台 Global 物理机装系统

**节点：** 每台 Global 物理机 BMC / Live ISO。不用 kubectl。

对照 `config.local.yaml` 的 WWN：

```text
<GLOBAL_CP_1_OS_DISK_WWN>
<GLOBAL_CP_2_OS_DISK_WWN>
<GLOBAL_CP_3_OS_DISK_WWN>
```

每台机器各自执行：

```bash
lsblk -d -e7 -o NAME,PATH,SIZE,MODEL,SERIAL,WWN,HCTL
ln -sfn /dev/disk/by-id/wwn-<这台机器的 OS 盘 WWN> \
  /dev/elemental-install-target
readlink -f /dev/elemental-install-target
test -b "$(readlink -f /dev/elemental-install-target)"
```

不要链到 ISO、loop 或数据盘。装完：弹出虚拟光驱，BMC 改成硬盘优先，等机器回到网络。

回到 CAN 节点：

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

三个 inventory `Ready=True` 后，把实际名称填进 `config.local.yaml` 的：

```text
<GLOBAL_CP_INVENTORY_1>
<GLOBAL_CP_INVENTORY_2>
<GLOBAL_CP_INVENTORY_3>
```

三个名称必须互不相同。

### 4.3 `global/control-plane/02-control-plane-pool.yaml`

| 变量 | 字段 |
|---|---|
| `<GLOBAL_CP_INVENTORY_1>` | `spec.machineInventories[0].name` |
| `<GLOBAL_CP_INVENTORY_2>` | `spec.machineInventories[1].name` |
| `<GLOBAL_CP_INVENTORY_3>` | `spec.machineInventories[2].name` |
| `<NETWORK_DEVICE>` | 三个条目的 `networkDevice` |

hostname 保持 `global-cp-1` / `global-cp-2` / `global-cp-3`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/02-control-plane-pool.yaml"
```

### 4.4 `global/control-plane/03-baremetal-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `metadata.annotations.cpaas.io/registry-address`，首次创建写成 `<BOOTSTRAP_HOST_IP>:11443` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `spec.controlPlaneLoadBalancer.host` |
| `<NETWORK_DEVICE>` | `spec.networkDevice` |

注释里的 `<PLATFORM_REGISTRY>` 只用于 installer 成功后的 patch 目标，首次 apply 不要改成平台仓库。`type: External`、`port: 6443` 不要改。installer 成功后如果这个对象还停在 Bootstrap 仓库，只 patch annotation 到 `<PLATFORM_REGISTRY>:11443`，不要重新 apply 整个文件。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/03-baremetal-cluster.yaml"
```

### 4.5 `global/control-plane/04-control-plane-machine-template.yaml`

无变量。确认 `machineInventoryPoolRef.name` 是 `global-control-plane-pool`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/04-control-plane-machine-template.yaml"
```

### 4.6 `global/control-plane/05-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `metadata.annotations.cpaas.io/registry-address`，首次创建写成 `<BOOTSTRAP_HOST_IP>:11443` |
| `<GLOBAL_KUBE_OVN_JOIN_CIDR>` | `metadata.annotations.cpaas.io/kube-ovn-join-cidr` |
| `<GLOBAL_POD_CIDR>` | `spec.clusterNetwork.pods.cidrBlocks[0]` |
| `<GLOBAL_SERVICE_CIDR>` | `spec.clusterNetwork.services.cidrBlocks[0]` |

注释里的 `<PLATFORM_REGISTRY>` 是 installer 改写后的目标，首次 apply 不要改成平台仓库。Kubernetes 版本保持 `v1.35.6-acp.9`，Kube-OVN 保持 `v4.4.12`。installer 会改这个 Cluster 的 registry annotation，成功后不要重新 apply 这个文件。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/05-cluster.yaml"
```

### 4.7 `global/control-plane/06-kubeadm-control-plane.yaml`

| 变量 | 字段 |
|---|---|
| `<SSH_AUTHORIZED_KEY>` | `spec.kubeadmConfigSpec.users[0].sshAuthorizedKeys[0]` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `clusterConfiguration.etcd.local.serverCertSANs` |
| `<BOOTSTRAP_HOST_IP>` | 同上 `serverCertSANs` |

固定值：

```text
replicas     3
maxSurge     0
version      v1.35.6-acp.9
dns imageTag 1.14.7-v4.4.7
etcd imageTag v3.6.13-acp.2
```

kubelet patch 从 `protectKernelDefaults` 直接到 `tlsCertFile`，不要出现 `staticPodPath`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/06-kubeadm-control-plane.yaml"
kubectl -n cpaas-system get cluster global
kubectl -n cpaas-system get kubeadmcontrolplane global-kcp
kubectl -n cpaas-system get machine,baremetalmachine -o wide
```

三台 Global 控制面 Ready 后再做 handoff。

---

## 5. Global：handoff 和 installer

**节点：** CAN / Bootstrap。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

### 5.1 `global/bootstrap/04-dcs-import-extra-resources.yaml`

先读实际 inventory：

```bash
kubectl -n cpaas-system get baremetalmachine \
  -l cluster.x-k8s.io/cluster-name=global \
  -o custom-columns='NAME:.metadata.name,INVENTORY:.status.machineInventoryRef.name'
```

| 变量 | 字段 |
|---|---|
| `<GLOBAL_CP_INVENTORY_1>` | `machineinventories.elemental.cattle.io` names[0] |
| `<GLOBAL_CP_INVENTORY_2>` | names[1] |
| `<GLOBAL_CP_INVENTORY_3>` | names[2] |

`global-registration` 已写死，不要改。不要往这个文件加 SeedImage、token、plan、bootstrap TLS。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/04-dcs-import-extra-resources.yaml"
```

### 5.2 `global/bootstrap/05-installer-config.example.json`

复制到本机，不要改仓库里的 example：

```bash
cp "$PACK_DIR/global/bootstrap/05-installer-config.example.json" \
  /root/installer-config.json
```

| 变量 | JSON 字段 |
|---|---|
| `admin@cpaas.io` | `basic.username`，不要改 |
| `<BASE64_PLATFORM_ADMIN_PASSWORD>` | `basic.password` |
| `<PLATFORM_REGISTRY>` | `registry.domain`，写成 `<PLATFORM_REGISTRY>:11443` |
| `Registry` | `registry.username`，不要写成 `admin` |
| `<BASE64_REGISTRY_PASSWORD>` | `registry.password` |
| `<PLATFORM_HOST>` | `console.host[0]`、`console.globalHost` |
| `<GLOBAL_POD_CIDR>` | `cluster.clusterCIDR` |
| `<GLOBAL_SERVICE_CIDR>` | `cluster.serviceCIDR` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `cluster.features.ha.vip` |
| `<BOOTSTRAP_HOST_IP>` | `hostIP` |

`deployMode` 保持 `normal`。`features.ha` 是 installer 收 API VIP 的字段。

```bash
kubectl -n cpaas-system get pods -l service_name=cpaas-installer -o wide
export INSTALLER_IP=$(kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o jsonpath='{.items[0].status.podIP}')
test -n "$INSTALLER_IP"
```

一个终端跟日志：

```bash
kubectl -n cpaas-system logs -f \
  -l service_name=cpaas-installer --all-containers=true --prefix=true
```

另一个终端提交。必须打 installer Pod IP 的 `8080`，不要打平台 VIP：

```bash
curl -fS -X POST \
  "http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs" \
  -H 'Content-Type: application/json' \
  --data-binary @/root/installer-config.json
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
```

installer 成功后保存 `global-kubeconfig`。

---

## 6. Global：Finished provider 和 catalog CM

**节点：** CAN / 运维机。从此改用：

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl get nodes
```

### 6.1 确认 registry 已切到平台仓库

```bash
kubectl -n cpaas-system get cluster global \
  -o jsonpath='{.metadata.annotations.cpaas\.io/registry-address}'; echo
kubectl -n cpaas-system get baremetalcluster global \
  -o jsonpath='{.metadata.annotations.cpaas\.io/registry-address}'; echo
```

`BaremetalCluster/global` 如果还是 `<BOOTSTRAP_HOST_IP>:11443`，只 patch 到 `<PLATFORM_REGISTRY>:11443`。不要重新 apply `03` 或 `05`。

### 6.2 `global/control-plane/07-finished-kubeadm-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<KUBEADM_PROVIDER_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<PLATFORM_REGISTRY>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<PLATFORM_HOST>` | `values.global.host`、`values.global.platformUrl`（`https://<PLATFORM_HOST>`） |

这个文件必须 apply 到 Finished Global，不要 apply 到 `minialauda`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/07-finished-kubeadm-provider-apprelease.yaml"
```

### 6.3 `global/control-plane/08-finished-baremetal-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<PLATFORM_REGISTRY>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<PLATFORM_HOST>` | `values.global.host`、`values.global.platformUrl`（`https://<PLATFORM_HOST>`）、`values.elemental.server.url`（`https://<PLATFORM_HOST>`，不要加 `:12443`） |
| `<K25_AMD64_IMAGE_TAG>` | amd64 catalog tag |
| `<K25_ARM64_IMAGE_TAG>` | arm64 catalog tag |

这里必须同时有两个 catalog key，tag 不能交叉。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/08-finished-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
```

### 6.4 检查 ConfigMap `cpaas-system/elemental-image-catalog`

AppRelease 只是输入。Workload 真正读的是这个 CM：

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

CM 对象不存在时，先查 `08` 是否 Ready。两个 key 都对了，再创建 Workload。

---

# 第二部分：创建 Workload 集群

本章全部使用 `$GLOBAL_KUBECONFIG`。

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
```

---

## 7. Workload：amd64 控制面

### 7.1 `workload/control-plane/01-registration-seedimage-amd64.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `system-uri`、`baseImage` 的仓库地址，写成 `<PLATFORM_REGISTRY>:11443` |
| `<K25_AMD64_IMAGE_TAG>` | `system-uri`、`baseImage` 的 tag |

固定值：

```text
device          /dev/elemental-install-target
targetPlatform  linux/amd64
os id           alaudaos.k.25
```

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/01-registration-seedimage-amd64.yaml"
kubectl -n cpaas-system get seedimage hcs-workloadcluster-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

这个 ISO 只给三台 Workload amd64 控制面。不要复用 Global ISO。

### 7.2 三台 Workload 控制面物理机装系统

**节点：** 每台 Workload amd64 控制面 BMC / Live ISO。

WWN 对照：

```text
<WORKLOAD_CP_1_OS_DISK_WWN>
<WORKLOAD_CP_2_OS_DISK_WWN>
<WORKLOAD_CP_3_OS_DISK_WWN>
```

symlink 流程与 4.2 相同。装完回到 CAN：

```bash
kubectl -n cpaas-system get machineinventory \
  -o custom-columns='NAME:.metadata.name,READY:.status.ready'
```

把三个互不相同的实际名称填进：

```text
<WORKLOAD_CP_INVENTORY_1>
<WORKLOAD_CP_INVENTORY_2>
<WORKLOAD_CP_INVENTORY_3>
```

### 7.3 `workload/control-plane/02-control-plane-pool.yaml`

| 变量 | 字段 |
|---|---|
| `<WORKLOAD_CP_INVENTORY_1>` | `spec.machineInventories[0].name` |
| `<WORKLOAD_CP_INVENTORY_2>` | `spec.machineInventories[1].name` |
| `<WORKLOAD_CP_INVENTORY_3>` | `spec.machineInventories[2].name` |
| `<NETWORK_DEVICE>` | 三个条目的 `networkDevice` |

hostname 保持 `hcs-workloadcluster-cp-1/2/3`。三个 name 不能重复。如果 API 报 `spec.machineInventories[2].name: Duplicate value`，只改第三个 name，不要删 Cluster。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/02-control-plane-pool.yaml"
```

### 7.4 `workload/control-plane/03-baremetal-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `metadata.annotations.cpaas.io/registry-address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | `spec.controlPlaneLoadBalancer.host` |
| `<NETWORK_DEVICE>` | `spec.networkDevice` |

不要写成 Bootstrap 仓库或 `127.0.0.1`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/03-baremetal-cluster.yaml"
```

### 7.5 `workload/control-plane/04-control-plane-machine-template.yaml`

无变量。确认 `machineInventoryPoolRef.name` 是 `hcs-workloadcluster-control-plane-pool`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/04-control-plane-machine-template.yaml"
```

### 7.6 `workload/control-plane/05-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `metadata.annotations.cpaas.io/registry-address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<WORKLOAD_KUBE_OVN_JOIN_CIDR>` | `metadata.annotations.cpaas.io/kube-ovn-join-cidr` |
| `<WORKLOAD_POD_CIDR>` | `spec.clusterNetwork.pods.cidrBlocks[0]` |
| `<WORKLOAD_SERVICE_CIDR>` | `spec.clusterNetwork.services.cidrBlocks[0]` |

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/05-cluster.yaml"
```

### 7.7 `workload/control-plane/06-kubeadm-control-plane.yaml`

| 变量 | 字段 |
|---|---|
| `<SSH_AUTHORIZED_KEY>` | `spec.kubeadmConfigSpec.users[0].sshAuthorizedKeys[0]` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | `clusterConfiguration.etcd.local.serverCertSANs` |

固定值：`replicas: 3`、`maxSurge: 0`、`version: v1.35.6-acp.9`。不要写 `staticPodPath`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/06-kubeadm-control-plane.yaml"
kubectl get cluster hcs-workloadcluster
kubectl get kubeadmcontrolplane hcs-workloadcluster-control-plane
kubectl get machines -l cluster.x-k8s.io/cluster-name=hcs-workloadcluster -o wide
```

---

## 8. Workload：arm64 Worker

### 8.1 `workload/worker/01-registration-seedimage-arm64.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `system-uri`、`baseImage` 的仓库地址，写成 `<PLATFORM_REGISTRY>:11443` |
| `<K25_ARM64_IMAGE_TAG>` | `system-uri`、`baseImage` 的 tag |

固定值：

```text
device          /dev/elemental-install-target
targetPlatform  linux/arm64
os id           alaudaos.k.25
```

不要把 amd64 tag 或 amd64 ISO 用到这台机器。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/01-registration-seedimage-arm64.yaml"
kubectl -n cpaas-system get seedimage \
  hcs-workloadcluster-arm64-worker-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

### 8.2 ARM 物理机装系统

**节点：** 那一台 ARM Worker BMC / Live ISO。

WWN 对照 `<WORKLOAD_ARM64_WORKER_1_OS_DISK_WWN>`。symlink 流程与 4.2 相同。

装完后把实际 inventory 名称填进 `<WORKLOAD_ARM64_WORKER_INVENTORY>`。

### 8.3 `workload/worker/02-worker-pool-arm64.yaml`

| 变量 | 字段 |
|---|---|
| `<WORKLOAD_ARM64_WORKER_INVENTORY>` | `spec.machineInventories[0].name` |
| `<NETWORK_DEVICE>` | `networkDevice` |

hostname 保持 `hcs-workloadcluster-arm64-worker-1`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/02-worker-pool-arm64.yaml"
```

### 8.4 `workload/worker/03-worker-machine-template-arm64.yaml`

无变量。确认 `machineInventoryPoolRef.name` 是 `hcs-workloadcluster-arm64-worker-pool`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/03-worker-machine-template-arm64.yaml"
```

### 8.5 `workload/worker/04-worker-kubeadm-config-template-arm64.yaml`

| 变量 | 字段 |
|---|---|
| `<SSH_AUTHORIZED_KEY>` | `spec.template.spec.users[0].sshAuthorizedKeys[0]` |

固定值必须保留：

```text
volume-plugin-dir: /opt/libexec/kubernetes/kubelet-plugins/volume/exec/
```

kubelet patch 不要写 `staticPodPath`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/04-worker-kubeadm-config-template-arm64.yaml"
```

### 8.6 `workload/worker/05-worker-machine-deployment-arm64.yaml`

无变量。固定值：

```text
replicas        1
maxSurge        0
maxUnavailable  1
version         v1.35.6-acp.9
```

```bash
kubectl apply -f "$PACK_DIR/workload/worker/05-worker-machine-deployment-arm64.yaml"
kubectl get machines -l cluster.x-k8s.io/cluster-name=hcs-workloadcluster -o wide
```

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

漏做了对时：先把相关物理机时间同步好，再删卡住的 `Machine`，看控制器会不会重建。

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

`ImageCatalogMiss`：先按 6.4 补 catalog CM，再只删失败的 `BaremetalMachine`。

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

pool 也被清掉时，先确认 inventory Ready，再 apply 两个 `02`。已经存在的 Registration / SeedImage / inventory 不要重复创建。不要删 `Cluster/global`。

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
