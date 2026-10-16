# HCS Bare Metal / ACP 4.4.1 / AlaudaOS K25

这是一个面向固定 HCS 环境的 **YAML-only、单站点、非容灾** Bare Metal 交付包。

本 README 按实际执行顺序说明：在哪个节点执行命令、使用哪个 kubeconfig、Apply 哪个文件，以及每个阶段需要修改哪些占位符。

固定拓扑：

- Global：3 个 amd64/x86 控制面。
- Workload 控制面：3 个 amd64/x86 节点。
- Workload Worker：1 个 arm64 节点。
- 首次交付不部署 Global Worker。
- 所有 Bare Metal 控制面和 Worker 的 `maxSurge` 都是 `0`。
- 所有安装 YAML 都使用 `/dev/elemental-install-target`。

三个控制面节点是各自集群内部的 Kubernetes 控制面数量，不代表第二站点或 ACP 灾备拓扑。

---

## 0. 节点、文件和 kubeconfig 对照

| 阶段 | 命令执行位置 | kubeconfig / 连接方式 | 目标 |
|---|---|---|---|
| Bootstrap | Bootstrap 节点 | Bootstrap kubeconfig，通常连接 KIND `minialauda` | 安装 provider、创建 Global ISO、创建 Global 控制面 |
| Live ISO 安装 | 每台物理机的 BMC/虚拟控制台和 Live ISO shell | 不使用 kubeconfig | 将 ISO 安装到正确 OS 磁盘 |
| Installer handoff | Bootstrap 节点 | Bootstrap kubeconfig；curl 访问 installer Pod IP | 将 Global 从 bootstrap registry 切换到平台环境 |
| Finished Global | Bootstrap 节点或运维机 | `global-kubeconfig` | 安装 finished provider、创建 Workload 资源 |
| Workload 主机安装 | 每台 Workload 物理机的 BMC/Live ISO shell | 不使用 kubeconfig | 安装 Workload amd64/arm64 OS |

建议先设置本地变量。以下只是 shell 变量，不是仓库脚本：

```bash
export PACK_DIR=/path/to/baremetal-hcs
export BOOTSTRAP_KUBECONFIG=/path/to/bootstrap-kubeconfig
export GLOBAL_KUBECONFIG=/path/to/global-kubeconfig
```

使用 `kubectl` 时明确指定 kubeconfig，避免把资源 Apply 到错误的集群：

```bash
kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" get nodes
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" get nodes
```

---

## 1. 目录和唯一配置文件

```text
config/
  config.example.yaml

global/
  bootstrap/
  control-plane/

workload/
  control-plane/
  worker/
```

没有脚本、renderer、Kustomize overlay 或 Helm generator。

只填写一个文件：

```text
config/config.example.yaml
```

复制为本地文件：

```bash
cd "$PACK_DIR"
cp config/config.example.yaml config/config.local.yaml
```

`config/config.local.yaml` 已被 `.gitignore` 忽略。仓库没有 renderer，修改该文件不会自动修改 manifests；需要把相同的占位符手工替换到编号 YAML/JSON 中。

### 1.1 必须先修改的配置项

| 配置区域 | 需要填写的内容 | 作用 |
|---|---|---|
| `release` | ACP package、provider package/chart、amd64/arm64 archive、最终 image tag、checksum | 确认实际交付物，不使用旧 preview 值 |
| `bootstrap` | `<BOOTSTRAP_HOST_IP>`、`<KIND_REGISTRY_ADDRESS>` | Bootstrap 节点、KIND registry 和 bootstrap registry |
| `platform` | `<PLATFORM_HOST>`、`<PLATFORM_REGISTRY>`、Global/Workload VIP、`<NETWORK_DEVICE>` | Finished Global 和两个集群的网络入口 |
| `network` | Global/Workload Pod、Service、Kube-OVN join CIDR | 写入 Cluster 和 installer JSON |
| `ssh` | `<SSH_AUTHORIZED_KEY>` | 写入 KCP 和 Worker bootstrap 配置 |
| `inventory` | Global CP、Workload CP、ARM Worker 的 inventory 名称 | 写入 MachineInventoryPool 和 DCS handoff |
| `hosts` | 每台机器 OS disk WWN | 只用于现场核对，不写入共享 YAML |
| `installer` | Registry 用户名 `Registry`、registry 密码、平台管理员密码 | 只填写本地 installer JSON，不提交密码 |

Registry 用户名固定为：

```text
Registry
```

平台基础账号 `admin@cpaas.io` 是平台管理员账号，不是 Registry 账号；不要把它填入 installer JSON 的 `registry.username`。

### 1.2 需要手工同步到 manifests 的文件

| 阶段 | 文件 | 需要重点确认/替换 |
|---|---|---|
| Bootstrap provider | `global/bootstrap/02-kubeadm-provider-apprelease.yaml` | KIND registry、Kubeadm provider chart revision |
| Bootstrap Bare Metal provider | `global/bootstrap/03-baremetal-provider-apprelease.yaml` | KIND registry、Bare Metal provider revision、amd64 K25 rootfs tag |
| Global Registration/ISO | `global/control-plane/01-registration-seedimage-amd64.yaml` | bootstrap rootfs/ISO registry、amd64 tag、`linux/amd64` |
| Global inventory | `global/control-plane/02-control-plane-pool.yaml` | 3 个 Global inventory、`<NETWORK_DEVICE>` |
| Global first-create | `global/control-plane/03-baremetal-cluster.yaml`、`05-cluster.yaml` | Global VIP、bootstrap registry、Global CIDR |
| Global KCP | `global/control-plane/04-control-plane-machine-template.yaml`、`06-kubeadm-control-plane.yaml` | SSH key、KCP 版本、`replicas: 3`、`maxSurge: 0` |
| Finished Global provider | `global/control-plane/07-finished-kubeadm-provider-apprelease.yaml`、`08-finished-baremetal-provider-apprelease.yaml` | 平台 registry、平台 host、amd64/arm64 catalog 和 tag |
| Handoff | `global/bootstrap/04-dcs-import-extra-resources.yaml` | 实际 Global inventory 名称 |
| Installer | `global/bootstrap/05-installer-config.example.json` | Registry 用户名 `Registry`、密码、平台 host、Global VIP、Global CIDR |
| Workload amd64 | `workload/control-plane/01`–`06` | 平台 registry、Workload inventory/VIP/CIDR、SSH key |
| Workload arm64 | `workload/worker/01`–`05` | ARM tag、`linux/arm64`、ARM inventory、SSH key、K25 plugin directory |

---

## 2. ACP 4.4.1 交付物门禁

固定版本：

```text
Kubernetes  v1.35.6-acp.9
CoreDNS     1.14.7-v4.4.7
etcd        v3.6.13-acp.2
containerd  2.2.8-3
pause       3.10
Kube-OVN    v4.4.12
```

K25 catalog key：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25
v1.35.6-acp.9_arm64_alaudaos.k.25
```

部署前必须确认 hybrid Core Package、`ALL` Kubeadm provider、支持 KubeOS/amd64/arm64 的 `ALL` Bare Metal provider，以及 amd64/arm64 KubeOS archive、checksum 和最终 image tag 来自同一套 ACP 4.4.1 交付物。

### 2.1 Catalog、rootfs 和 ISO 的替换关系

本包中有三种不同的镜像引用，不能混用：

| 用途 | 镜像 | 读取位置 |
|---|---|---|
| Bare Metal catalog | `tkestack/kubeos-rootfs:<K25_*_IMAGE_TAG>` | Bare Metal provider AppRelease 的 `provider.imageCatalog.images` |
| 节点安装系统 | `tkestack/kubeos-rootfs:<K25_*_IMAGE_TAG>` | MachineRegistration 的 `spec.config.elemental.install.system-uri` |
| 生成 Live ISO | `tkestack/kubeos-iso:<K25_*_IMAGE_TAG>` | SeedImage 的 `spec.baseImage` |

最终 image tag 不是 `baremetal`。`baremetal` 只是 archive 导入后的本地 source tag；推送后必须使用 Customer Portal archive 对应的最终 tag。

#### Bootstrap 阶段：只替换 amd64 catalog

文件：

```text
global/bootstrap/03-baremetal-provider-apprelease.yaml
```

保持 catalog key 不变，只替换 repository/tag 中的最终值：

```yaml
provider:
  imageCatalog:
    images:
      v1.35.6-acp.9_amd64_alaudaos.k.25:
        repository: tkestack/kubeos-rootfs
        tag: <K25_AMD64_IMAGE_TAG>
```

Bootstrap provider 只需要这个 amd64 catalog，因为它只负责创建 Global amd64 控制面。

#### Finished Global 阶段：同时替换 amd64 和 arm64 catalog

文件：

```text
global/control-plane/08-finished-baremetal-provider-apprelease.yaml
```

必须同时存在以下两个条目：

```yaml
provider:
  imageCatalog:
    images:
      v1.35.6-acp.9_amd64_alaudaos.k.25:
        repository: tkestack/kubeos-rootfs
        tag: <K25_AMD64_IMAGE_TAG>
      v1.35.6-acp.9_arm64_alaudaos.k.25:
        repository: tkestack/kubeos-rootfs
        tag: <K25_ARM64_IMAGE_TAG>
```

这里不能把 ARM tag 写成 amd64 tag，也不能把 `arm64` catalog key 改成 `amd64`。ARM tag 的后缀必须以实际 archive 为准，不能凭空猜测 `.aarch64`。

#### Registration/SeedImage 必须与 catalog 对应

Global amd64：

```text
global/control-plane/01-registration-seedimage-amd64.yaml
system-uri: <BOOTSTRAP_HOST_IP>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
baseImage: <KIND_REGISTRY_ADDRESS>:11443/tkestack/kubeos-iso:<K25_AMD64_IMAGE_TAG>
```

Workload amd64：

```text
workload/control-plane/01-registration-seedimage-amd64.yaml
system-uri: <PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
baseImage: <PLATFORM_REGISTRY>:11443/tkestack/kubeos-iso:<K25_AMD64_IMAGE_TAG>
```

Workload arm64：

```text
workload/worker/01-registration-seedimage-arm64.yaml
system-uri: <PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>
baseImage: <PLATFORM_REGISTRY>:11443/tkestack/kubeos-iso:<K25_ARM64_IMAGE_TAG>
targetPlatform: linux/arm64
```

检查顺序必须是：

1. archive 架构正确；
2. archive 中的最终 rootfs/ISO tag 已确认；
3. 两个 image 已推送到对应 registry；
4. provider AppRelease 的 catalog key/tag 已替换；
5. Registration 的 `system-uri` 使用同架构 rootfs tag；
6. SeedImage 的 `baseImage` 使用同架构 ISO tag；
7. 最后才创建或重建 Machine。

---

## 3. 导入并推送 KubeOS image

### 3.1 在 Bootstrap 节点导入 amd64 archive

命令在 **Bootstrap 节点**执行。先按站点安全策略配置 registry trust；不要关闭 TLS 校验。

两个架构 archive 都使用相同的本地 source reference：

```text
kubeos-rootfs:baremetal
kubeos-iso:baremetal
```

一次只导入一个架构：

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

### 3.2 导入并推送 arm64 archive

在同一个 Bootstrap 节点重复执行，但先确认：

- archive 是 arm64；
- `OS_TAG=<K25_ARM64_IMAGE_TAG>`；
- 目标 tag 不会覆盖 amd64 tag。

Workload 创建前，平台 registry 必须同时存在 amd64 和 arm64 的 `kubeos-rootfs`、`kubeos-iso` image。

---

## 4. 创建 Global 集群：Bootstrap 阶段

本章的命令全部在 **Bootstrap 节点**执行，并使用：

```bash
export KUBECONFIG="$BOOTSTRAP_KUBECONFIG"
```

### 4.1 启动并检查 Bootstrap 集群

```bash
kubectl get nodes
kubectl get storageclass
kubectl -n cpaas-system get pods
```

必须确认：

- KIND 集群名称为 `minialauda`；
- `cpaas-system` 基础组件正常；
- 有默认 StorageClass；
- SeedImage 使用的 `20Gi` PVC 可以创建。

### 4.2 安装 Bootstrap provider

按顺序执行：

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/01-dex-tls-bootstrap.yaml"
kubectl apply -f "$PACK_DIR/global/bootstrap/02-kubeadm-provider-apprelease.yaml"
kubectl apply -f "$PACK_DIR/global/bootstrap/03-baremetal-provider-apprelease.yaml"
```

对应文件和修改点：

- `01-dex-tls-bootstrap.yaml`：确认 Bootstrap host IP 相关证书 SAN；不要把真实私钥提交到仓库。
- `02-kubeadm-provider-apprelease.yaml`：`spec.source.repoURL` 使用 KIND registry，`targetRevision` 使用实际 Kubeadm provider chart revision。
- `03-baremetal-provider-apprelease.yaml`：Bootstrap registry 使用 KIND registry；Bootstrap catalog 只保留 amd64 K25 catalog；`systemAgent` 和 TLS 配置保持现状。

检查 provider：

```bash
kubectl -n cpaas-system get apprelease
kubectl -n cpaas-system get pods -o wide
```

只有 provider Ready 后才进入 Registration/SeedImage 阶段。

### 4.3 创建 Global amd64 Registration 和 ISO

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/01-registration-seedimage-amd64.yaml"
kubectl -n cpaas-system get machineregistration global-registration
kubectl -n cpaas-system get seedimage global-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

文件：

```text
global/control-plane/01-registration-seedimage-amd64.yaml
```

必须确认：

- `os.baremetal.alauda.io/id: alaudaos.k.25`；
- `os.baremetal.alauda.io/arch: "${Runtime/Architecture}"`；
- `system-uri` 使用 amd64 `kubeos-rootfs`；
- `baseImage` 使用 amd64 `kubeos-iso`；
- `targetPlatform: linux/amd64`；
- `device: /dev/elemental-install-target`；
- `status.downloadURL` 非空后才下载 ISO。

---

## 5. 在三台 Global 物理机安装 KubeOS

这一步不在 Kubernetes 节点上执行 `kubectl`。在每台 Global 物理机的 BMC/虚拟控制台启动对应 ISO，然后进入 Live ISO shell。

三台机器必须使用同一个 Global Registration 生成的 ISO 内容，但每台机器都要建立自己的 OS disk symlink。

### 5.1 每台机器现场识别 OS 磁盘

```bash
lsblk -d -e7 -o NAME,PATH,SIZE,MODEL,SERIAL,WWN,HCTL
```

根据 `config/config.local.yaml` 中对应的 WWN，确认这台主机的目标 OS 盘。不要凭经验使用 `/dev/sda`。

### 5.2 每台机器创建安装设备 symlink

```bash
ln -sfn /dev/disk/by-id/wwn-<THIS_HOST_OS_DISK_WWN> \
  /dev/elemental-install-target
readlink -f /dev/elemental-install-target
test -b "$(readlink -f /dev/elemental-install-target)"
```

确认目标不是 ISO、loop device 或数据盘，然后从 BMC 挂载并启动对应 SeedImage ISO。

安装完成后：

1. eject virtual media；
2. 将 BMC boot order 调整为 disk-first；
3. 等待主机重启并回到网络；
4. 回到 Bootstrap 节点检查 MachineInventory。

### 5.3 创建 Global inventory pool

回到 **Bootstrap 节点**，仍使用 Bootstrap kubeconfig：

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/02-control-plane-pool.yaml"
```

文件：

```text
global/control-plane/02-control-plane-pool.yaml
```

替换：

- `<GLOBAL_CP_INVENTORY_1>`；
- `<GLOBAL_CP_INVENTORY_2>`；
- `<GLOBAL_CP_INVENTORY_3>`；
- `<NETWORK_DEVICE>`。

检查 inventory：

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

### 5.4 创建 Global BaremetalCluster、Cluster 和 KCP

按顺序执行：

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/03-baremetal-cluster.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/04-control-plane-machine-template.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/05-cluster.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/06-kubeadm-control-plane.yaml"
```

对应修改点：

- `03-baremetal-cluster.yaml`：Global VIP、`<NETWORK_DEVICE>`、首次创建时的 Bootstrap registry。
- `04-control-plane-machine-template.yaml`：引用 `global-control-plane-pool`。
- `05-cluster.yaml`：Global Pod/Service/join CIDR、Kubernetes 版本、Bootstrap registry。
- `06-kubeadm-control-plane.yaml`：SSH key、`replicas: 3`、`maxSurge: 0`、Kubernetes/Kube-OVN/CoreDNS/etcd 版本。
- 所有 kubelet patch 都不能出现 `staticPodPath`。

检查创建过程：

```bash
kubectl -n cpaas-system get cluster global
kubectl -n cpaas-system get kubeadmcontrolplane global-kcp
kubectl -n cpaas-system get baremetalmachine -o wide
kubectl get nodes -o wide
```

等待三台 Global 控制面节点和 `global-kcp` Ready 后再进入 handoff。

---

## 6. Global handoff：导入资源和执行 installer

### 6.1 填写 DCS handoff inventory

先在 Bootstrap 节点读取实际 inventory 名称：

```bash
kubectl -n cpaas-system get baremetalmachine \
  -l cluster.x-k8s.io/cluster-name=global \
  -o custom-columns='NAME:.metadata.name,INVENTORY:.status.machineInventoryRef.name'
```

将实际 inventory 名称填入：

```text
global/bootstrap/04-dcs-import-extra-resources.yaml
```

必须对应：

- `<GLOBAL_CP_INVENTORY_1>`；
- `<GLOBAL_CP_INVENTORY_2>`；
- `<GLOBAL_CP_INVENTORY_3>`；
- `global-registration`。

本文件不导入 SeedImage、registration token、Elemental plan、bootstrap TLS 或 kubeadm bootstrap Secret。

### 6.2 Apply handoff ConfigMap

仍在 Bootstrap 节点执行：

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/04-dcs-import-extra-resources.yaml"
```

### 6.3 创建本地 installer JSON

不要直接修改或提交仓库里的示例文件。复制到 Bootstrap 节点的 gitignored 路径：

```bash
cp "$PACK_DIR/global/bootstrap/05-installer-config.example.json" \
  /root/installer-config.json
```

编辑 `/root/installer-config.json`，确认：

- `registry.domain` 为 `<PLATFORM_REGISTRY>:11443`；
- `registry.username` 为 `Registry`；
- `registry.password` 为实际 registry 密码的 base64 值；
- `console.host` 和 `globalHost` 为平台 host；
- `cluster.features.ha.vip` 为 Global API VIP；
- `cluster.clusterCIDR` 和 `serviceCIDR` 与 Global Cluster YAML 使用相同值；
- `deployMode` 为 `normal`；
- `hostIP` 为 Bootstrap host IP。

`features.ha` 是 installer 接收 Kubernetes API VIP 的固定接口字段，不是灾备配置。

### 6.4 查看 installer 并提交 handoff

仍然从 **Bootstrap 节点**执行：

```bash
kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o wide

export INSTALLER_IP=$(kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o jsonpath='{.items[0].status.podIP}')
test -n "$INSTALLER_IP"
```

实时查看日志：

```bash
kubectl -n cpaas-system logs -f \
  -l service_name=cpaas-installer \
  --all-containers=true --prefix=true
```

提交 installer：

```bash
curl -fS -X POST \
  "http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs" \
  -H 'Content-Type: application/json' \
  --data-binary @/root/installer-config.json
```

查询进度：

```bash
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
```

如果 installer 重启，查看之前容器的日志：

```bash
kubectl -n cpaas-system logs \
  -l service_name=cpaas-installer --previous --all-containers=true
```

不要把 POST 请求发到平台 VIP，也不要发到 `https://<PLATFORM_HOST>/...`；必须从 Bootstrap 节点发到 installer Pod IP 的 `8080` 端口。

---

## 7. Finished Global provider

Installer 成功后先取得并保护 `global-kubeconfig`，然后在 Bootstrap 节点或运维机设置：

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl get nodes
```

### 7.1 检查 registry handoff

```bash
kubectl -n cpaas-system get cluster global \
  -o jsonpath='{.metadata.annotations.cpaas\.io/registry-address}'; echo
kubectl -n cpaas-system get baremetalcluster global \
  -o jsonpath='{.metadata.annotations.cpaas\.io/registry-address}'; echo
```

如果 `BaremetalCluster/global` 仍是 Bootstrap registry，只 patch 这个对象到平台 registry。不要重新 Apply首次创建的 `global/control-plane/03` 或 `05`，否则可能覆盖运行中的 registry/LB 状态。

### 7.2 Apply finished provider

确认平台 registry 已经推送 provider charts 和 amd64/arm64 KubeOS images 后执行：

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/07-finished-kubeadm-provider-apprelease.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/08-finished-baremetal-provider-apprelease.yaml"
```

文件重点：

- `07` 使用平台 registry 和 Kubeadm provider chart revision。
- `08` 使用平台 registry和 Bare Metal provider revision。
- `08` 必须同时包含：
  - `v1.35.6-acp.9_amd64_alaudaos.k.25`
  - `v1.35.6-acp.9_arm64_alaudaos.k.25`
- 两个 catalog 必须使用各自架构的 rootfs tag。

检查 Ready：

```bash
kubectl -n cpaas-system get apprelease
kubectl -n cpaas-system get elementalimagecatalog -o wide
```

只有两个 catalog key 都可以解析并 Ready 后，才可以删除 Bootstrap KIND 集群 `minialauda`。

---

## 8. 创建 Workload 集群

本章全部使用 `global-kubeconfig`，在 Bootstrap 节点或运维机执行：

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
```

### 8.1 创建 Workload amd64 Registration 和 ISO

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/01-registration-seedimage-amd64.yaml"
kubectl -n cpaas-system get seedimage hcs-workloadcluster-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

文件：

```text
workload/control-plane/01-registration-seedimage-amd64.yaml
```

确认使用：

- 平台 registry；
- `<K25_AMD64_IMAGE_TAG>`；
- `targetPlatform: linux/amd64`；
- `/dev/elemental-install-target`。

下载 ISO 后，在三台 Workload amd64 控制面物理机上分别按照第 5 章的 WWN symlink 流程安装。不要复用 Global 控制面 ISO。

### 8.2 创建 Workload amd64 inventory pool

安装完成并 inventory Ready 后执行：

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/02-control-plane-pool.yaml"
kubectl -n cpaas-system get machineinventory -o wide
```

文件中替换：

- `<WORKLOAD_CP_INVENTORY_1>`；
- `<WORKLOAD_CP_INVENTORY_2>`；
- `<WORKLOAD_CP_INVENTORY_3>`；
- `<NETWORK_DEVICE>`。

### 8.3 创建 Workload Cluster 和控制面

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/03-baremetal-cluster.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/04-control-plane-machine-template.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/05-cluster.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/06-kubeadm-control-plane.yaml"
```

对应修改点：

- `03`：Workload VIP、平台 registry、`<NETWORK_DEVICE>`。
- `04`：引用 Workload control-plane pool。
- `05`：Workload Pod/Service/join CIDR、Kubernetes 版本和平台 registry。
- `06`：Workload SSH key、`replicas: 3`、`maxSurge: 0`、Kube-OVN/CoreDNS/etcd 版本。
- kubelet patch 不得出现 `staticPodPath`。

检查：

```bash
kubectl get cluster hcs-workloadcluster
kubectl get kubeadmcontrolplane hcs-workloadcluster-control-plane
kubectl get machines -l cluster.x-k8s.io/cluster-name=hcs-workloadcluster -o wide
```

### 8.4 创建 Workload arm64 Registration 和 ISO

```bash
kubectl apply -f "$PACK_DIR/workload/worker/01-registration-seedimage-arm64.yaml"
kubectl -n cpaas-system get seedimage \
  hcs-workloadcluster-arm64-worker-registration-iso \
  -o jsonpath='{.status.downloadURL}'; echo
```

文件必须确认：

- `system-uri` 使用 `<K25_ARM64_IMAGE_TAG>`；
- `baseImage` 使用 `<K25_ARM64_IMAGE_TAG>`；
- `targetPlatform: linux/arm64`；
- OS ID 为 `alaudaos.k.25`；
- install device 为 `/dev/elemental-install-target`。

下载 ARM ISO 后，只在目标 ARM Worker 物理机的 BMC/Live ISO shell 上安装。不要把 Workload amd64 ISO 给 ARM 主机使用。

### 8.5 创建 ARM-only Worker pool 和 MachineDeployment

ARM 主机完成注册并 Ready 后执行：

```bash
kubectl apply -f "$PACK_DIR/workload/worker/02-worker-pool-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/03-worker-machine-template-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/04-worker-kubeadm-config-template-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/05-worker-machine-deployment-arm64.yaml"
```

文件中确认：

- inventory 只有 `<WORKLOAD_ARM64_WORKER_INVENTORY>`；
- `networkDevice: <NETWORK_DEVICE>`；
- MachineTemplate、KubeadmConfigTemplate、MachineDeployment 使用 arm64 专用资源名；
- `replicas: 1`；
- `maxSurge: 0`；
- `maxUnavailable: 1`；
- kubelet 参数为：
  `/opt/libexec/kubernetes/kubelet-plugins/volume/exec/`。

检查最终状态：

```bash
kubectl get machines -l cluster.x-k8s.io/cluster-name=hcs-workloadcluster -o wide
kubectl get nodes -o wide
kubectl get machineinventory -o wide
```

---

## 9. 实际排障

### SeedImage 没有 `downloadURL`

检查默认 StorageClass、PVC、ISO repository/tag、registry 连通性、`targetPlatform` 和 Elemental Pod。成功门禁是 `status.downloadURL` 非空。

### `ImageCatalogMiss`

同时检查 Machine version、Inventory 的 K25 OS ID、Inventory architecture、完整 catalog key、rootfs tag 和 registry。修复 catalog 后只删除失败 Machine，让控制器重建；不要删除 Cluster。

### Installer 无响应或失败

确认：

1. curl 是从 Bootstrap 节点发起；
2. URL 使用 installer Pod IP，而不是平台 VIP；
3. 端口为 `8080`；
4. 路径为 `/cpaas-installer/api/config/dcs`；
5. 使用 `Content-Type: application/json`；
6. installer Pod 日志和 progress API 没有报错。

### ARM Worker 不启动

确认 ARM archive、ARM `system-uri`、ARM `baseImage`、`linux/arm64`、arm64 catalog key 和 ARM-only pool 没有混用 amd64 值。

### `staticPodPath`

所有 KCP 和 Worker kubelet patch 都完全省略 `staticPodPath`。不要写 `null`、空字符串或硬编码路径。

---

## 10. 安全约束和最终检查

部署前执行：

```bash
cd "$PACK_DIR"
find config global workload -type f | sort
ruby -e 'require "yaml"; Dir["**/*.yaml"].each { |f| YAML.load_stream(File.read(f)) }'
python3 -m json.tool global/bootstrap/05-installer-config.example.json >/dev/null
git diff --check
```

最终确认：

- `config/` 只有 `config.example.yaml`；
- 没有 `global/worker/`、`VERSION.yaml` 或旧 config 文件；
- 不提交 Registration URL、kubeconfig、密码、ISO、archive、私钥或真实 inventory 信息；
- Registry 用户名为 `Registry`，不是 `admin`；
- 不导入 bootstrap TLS、Elemental plan、registration token 或 kubeadm bootstrap Secret；
- 不关闭 registry TLS 验证；
- 不删除 `Cluster/global` 作为清理操作；
- 所有 KCP 和 Worker 都没有 `staticPodPath`；
- Global KCP 为 3 副本、Workload KCP 为 3 副本、ARM Worker 为 1 副本；
- 所有 Bare Metal rollout 的 `maxSurge` 为 `0`；
- 本包没有备用站点、容灾资源或故障切换配置。
