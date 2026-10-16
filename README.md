# HCS Bare Metal / ACP 4.4.1 / AlaudaOS K25

这是一个面向固定 HCS 环境的 **YAML-only、单站点、非容灾** Bare Metal 交付包。

本包只描述一条可执行部署路径，不提供备用站点、跨站点切换或其他拓扑选项。

固定拓扑：

- Global：3 个 amd64/x86 控制面。
- Workload 控制面：3 个 amd64/x86 节点。
- Workload Worker：1 个 arm64 节点。
- 首次交付不部署 Global Worker。
- 所有 Bare Metal 控制面和 Worker 的 `maxSurge` 都是 `0`。
- 所有安装 YAML 都使用 `/dev/elemental-install-target`。

三个控制面节点是各自集群内部的 Kubernetes 控制面数量，不代表第二站点或 ACP 灾备拓扑。

## 目录

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

## 1. 唯一配置对照文件

只填写并维护：

```text
config/config.example.yaml
```

该文件同时包含版本、Customer Portal 交付物、registry、VIP、CIDR、网络设备、SSH key、inventory、磁盘检查值和 installer 占位符。

复制为本地文件：

```bash
cp config/config.example.yaml config/config.local.yaml
```

`config/config.local.yaml` 已被 `.gitignore` 忽略。此仓库没有 renderer，编辑配置对照文件不会自动修改 manifests；必须把相同的 `<UPPERCASE_PLACEHOLDER>` 手工替换到编号 YAML/JSON 中。

WWN 只用于现场确认 OS 磁盘。共享 YAML 不写真实 WWN、`/dev/sda` 或其他固定设备名。

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

部署前必须从同一套 ACP 4.4.1 交付物确认 hybrid Core Package、`ALL` Kubeadm provider、支持 KubeOS/amd64/arm64 的 `ALL` Bare Metal provider，以及 amd64/arm64 KubeOS archive、checksum 和最终 image tag。

不能把旧 preview Kubernetes、旧 KubeOS tag 或旧 provider revision 当作 ACP 4.4.1 生产值。

## 3. KubeOS image 导入

两个架构 archive 都使用相同的本地 source reference：

```text
kubeos-rootfs:baremetal
kubeos-iso:baremetal
```

一次只导入一个架构。使用 `ctr`，不要使用 Docker push；registry trust 应按站点安全策略配置，不关闭 TLS 校验。

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
    -u 'admin:<REGISTRY_PASSWORD>' \
    "$REGISTRY/tkestack/$repo:$OS_TAG" \
    "$repo:baremetal"
done
```

对 ARM archive 重复时，必须确认 archive 是 arm64、tag 是 `<K25_ARM64_IMAGE_TAG>`，并推送到 ARM 对应的最终 tag。不要让两个架构覆盖同一个最终 tag。

Bootstrap 阶段至少需要 amd64 image；Workload 创建前，平台 registry 必须同时有 amd64 和 arm64 rootfs/ISO。

## 4. Bootstrap 和 Global

使用 bootstrap kubeconfig，在 bootstrap 集群上按编号执行：

1. 确认 KIND `minialauda` 和默认 StorageClass 可用，SeedImage PVC 可创建。
2. 将 provider charts 和 amd64 KubeOS image 放入 bootstrap registry。
3. Apply `global/bootstrap/01-dex-tls-bootstrap.yaml`。
4. Apply `global/bootstrap/02-kubeadm-provider-apprelease.yaml`。
5. Apply `global/bootstrap/03-baremetal-provider-apprelease.yaml`。
6. Apply `global/control-plane/01-registration-seedimage-amd64.yaml`，等待 `status.downloadURL` 非空。
7. 下载 Global 控制面 ISO；三台主机使用独立的 per-host Live ISO symlink。
8. 按顺序 Apply `global/control-plane/02` 至 `06`。
9. 等待三个 Global 控制面节点和 KCP Ready。
10. 用实际 `BaremetalMachine.status.machineInventoryRef.name` 替换 `global/bootstrap/04-dcs-import-extra-resources.yaml` 中的 inventory token，再 Apply。
11. 使用 installer JSON 从 bootstrap 节点 POST 到 `cpaas-installer` Pod IP 的 `:8080` API。

installer JSON 只是接口示例。复制到 gitignored 本地文件后再填写密码，不要提交填写后的文件。

installer JSON 中的 `features.ha` 是接收 Kubernetes API 外部 VIP 的固定接口字段，不是灾备开关；`deployMode: normal` 是本包固定部署模式。`dcs-import-extra-resources` 是 handoff 使用的历史资源名，不表示配置了灾备。

## 5. Installer handoff

```bash
kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o wide

kubectl -n cpaas-system logs -f \
  -l service_name=cpaas-installer \
  --all-containers=true --prefix=true
```

如果 installer 重启：

```bash
kubectl -n cpaas-system logs \
  -l service_name=cpaas-installer --previous --all-containers=true
```

查询进度：

```bash
export INSTALLER_IP=$(kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o jsonpath='{.items[0].status.podIP}')
test -n "$INSTALLER_IP"
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
```

Installer 成功后：

1. 保护 `global-kubeconfig`。
2. 检查 `Cluster/global` 和 `BaremetalCluster/global` 的 registry annotation。
3. 如果 `BaremetalCluster/global` 仍指向 bootstrap registry，只 patch 该对象；不要重新 Apply 首次创建文件覆盖运行中的状态。
4. 将 amd64/arm64 images 和 provider charts 推送到平台 registry。
5. 用 `global-kubeconfig` Apply `global/control-plane/07-finished-kubeadm-provider-apprelease.yaml` 和 `08-finished-baremetal-provider-apprelease.yaml`。
6. 确认两个 K25 catalog key Ready 后，再删除 KIND `minialauda`。

## 6. Workload 控制面和 ARM Worker

Finished Global Ready 后使用 `global-kubeconfig`。

### Workload amd64 控制面

按顺序 Apply：

```text
workload/control-plane/01-registration-seedimage-amd64.yaml
workload/control-plane/02-control-plane-pool.yaml
workload/control-plane/03-baremetal-cluster.yaml
workload/control-plane/04-control-plane-machine-template.yaml
workload/control-plane/05-cluster.yaml
workload/control-plane/06-kubeadm-control-plane.yaml
```

Workload 控制面使用独立的 amd64 Registration/ISO，不复用 Global 控制面 ISO。

### Workload arm64 Worker

按顺序 Apply：

```text
workload/worker/01-registration-seedimage-arm64.yaml
workload/worker/02-worker-pool-arm64.yaml
workload/worker/03-worker-machine-template-arm64.yaml
workload/worker/04-worker-kubeadm-config-template-arm64.yaml
workload/worker/05-worker-machine-deployment-arm64.yaml
```

ARM Worker 必须使用独立 ARM Registration/ISO、ARM rootfs/ISO、`targetPlatform: linux/arm64`、ARM-only InventoryPool，以及 `replicas: 1`、`maxSurge: 0`、`maxUnavailable: 1`。K25 kubelet plugin directory 为 `/opt/libexec/kubernetes/kubelet-plugins/volume/exec/`。

不要复用 Global 或 Workload amd64 控制面 ISO。

## 7. Live ISO 磁盘安全

每台主机现场确认 OS 盘后创建独立 symlink：

```bash
lsblk -d -e7 -o NAME,PATH,SIZE,MODEL,SERIAL,WWN,HCTL
ln -sfn /dev/disk/by-id/wwn-<THIS_HOST_OS_DISK_WWN> \
  /dev/elemental-install-target
readlink -f /dev/elemental-install-target
test -b "$(readlink -f /dev/elemental-install-target)"
```

确认目标不是 ISO、loop device 或数据盘；安装重启后 eject virtual media，并将 BMC boot order 调整为 disk-first。

## 8. 实际排障

### SeedImage 没有 `downloadURL`

检查默认 StorageClass、PVC、ISO repository/tag、registry 连通性、`targetPlatform` 和 Elemental Pod。成功门禁是 `status.downloadURL` 非空。

### `ImageCatalogMiss`

同时检查 Machine version、Inventory 的 K25 OS ID、Inventory architecture、完整 catalog key、rootfs tag 和 registry。修复 catalog 后只删除失败 Machine，让控制器重建；不要删除 Cluster。

### ARM Worker 不启动

确认 ARM archive、ARM `system-uri`、ARM `baseImage`、`linux/arm64`、arm64 catalog key 和 ARM-only pool 没有混用 amd64 值。

### `staticPodPath`

所有 KCP 和 Worker kubelet patch 都完全省略 `staticPodPath`。不要写 `null`、空字符串或硬编码路径。

## 9. 安全约束

- 不提交 Registration URL、kubeconfig、密码、ISO、archive、私钥或真实 inventory 信息。
- Base64 只是编码，不是加密。
- 不导入 bootstrap TLS、Elemental plan、registration token 或 kubeadm bootstrap Secret。
- 不关闭 registry TLS 验证。
- 不删除 `Cluster/global` 作为清理操作。
- 本包没有备用站点、容灾资源或故障切换配置。
