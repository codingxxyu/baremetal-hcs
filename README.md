# HCS Bare Metal ACP 4.4.1 交付手册

## 0. 交付范围

| 集群 | 角色 | 架构 | 数量 | 目录 |
|---|---|---|---|---|
| Global | 控制面 | amd64 | 3 | `global/` |
| Workload | 控制面 | amd64 | 3 | `workload/control-plane/` |
| Workload | Worker | arm64 | 1 | `workload/worker/` |

不部署 Global Worker。控制面和 Worker 的 `maxSurge` 一律 `0`。Cluster / BaremetalCluster 名必须是 `global`。Workload 集群名必须是 `hcs-workloadcluster`，不能叫 `global`。

主机怎么变成 Inventory，现场按机器类型选一种，**同一台机器不要两种都做**：

| | 路径 A：同网段裸金属 | 路径 B：HCS 虚拟机 |
|---|---|---|
| 操作系统从哪来 | SeedImage 生成 Live ISO，往空盘安装 | 研发 qcow2 模板，HCS 当系统镜像上传 |
| YAML | 现有 `01-registration-seedimage-*.yaml` | 新增的 `01-registration-amd64.yaml` / `01-registration-arm64.yaml`（不要 SeedImage） |
| 闸门 | `SeedImage.status.downloadURL` | `MachineRegistration.status.registrationURL`，再 curl 配置 |
| 启动 | BMC 挂 ISO，Live 上软链 `/dev/elemental-install-target` | HCS 用模板建虚机，高级选项「用户数据注入」贴 cloud-init |
| 安装设备 | `/dev/elemental-install-target` | 不用软链。系统盘默认 `/dev/vda`，启动必须 **UEFI** |

Inventory `Ready=True` 之后，Pool / Cluster / KCP / MD **两条路径相同**。ISO 文件全部保留；qcow2 只是多一份同名的 Registration，不要和 ISO 那份一起 apply。

几个名字在全文里固定指这些东西，没有别的缩写：

| 文档里的说法 | 是什么 |
|---|---|
| 引导节点 | 解 Core Package、跑 `setup.sh`、起 KIND 的那一台传统 OS 机器。IP 就是 `<BOOTSTRAP_HOST_IP>` |
| KIND / `minialauda` | `setup.sh` 在引导节点上拉起的引导集群，用来创建 Global。installer 交接完成后拆掉 |
| 成品 Global | 三台控制面装好、installer 成功之后的管理集群。之后 Workload 都 apply 到这里 |
| 引导仓库 | KIND 自带的仓库。本机 `127.0.0.1:11443`，KIND 里常见 `<KIND_REGISTRY_ADDRESS>:11443` |
| 平台仓库 | installer 起来之后的 `<PLATFORM_REGISTRY>:11443`，永久使用 |

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

`baremetal` 只是 archive 导入后的本地 source tag。YAML 里必须写成 archive 导入后读到的最终 tag：`<K25_AMD64_IMAGE_TAG>`、`<K25_ARM64_IMAGE_TAG>`。不要猜 ARM 后缀。

kubelet patch 不要写 `staticPodPath`，包括 `null`。

### 0.1 执行位置和 kubeconfig

第 2 章跑完 `setup.sh` 之后，引导集群的 kubeconfig 才存在。第 5.8 节 KCP Ready 之后，才能从 Secret 抽出 `global-kubeconfig`。进场时先只设包目录：

```bash
export PACK_DIR=/path/to/baremetal-hcs
```

| 阶段 | 执行节点 | kubeconfig | 目标集群 |
|---|---|---|---|
| 第 2–6 章 | 引导节点 | `$BOOTSTRAP_KUBECONFIG`（第 2 章生成） | KIND `minialauda` |
| 路径 A 装 OS | 对应物理机 BMC / Live ISO | 无 | 该物理机 |
| 路径 B 装 OS | HCS 控制台建虚机 | 无 | 该虚机；cloud-init 在虚机里跑 |
| 第 7 章起 | 引导节点 | `$GLOBAL_KUBECONFIG`（第 5.8 节抽出） | 成品 Global |
| 第 10 章 | 见各命令 | 见各命令 | 见各命令 |

### 0.2 对照文件

唯一对照文件：

```text
config/config.example.yaml
```

```bash
cp "$PACK_DIR/config/config.example.yaml" "$PACK_DIR/config/config.local.yaml"
```

每一步列出的变量，都从 `config.local.yaml` 取实际值，手工替换到该步 YAML / JSON。改对照文件不会自动改 manifests。apply 前检查目标文件：

```bash
grep -nE '<[^>]+>' 你准备 apply 的那个文件
```

文件里还留着 `<...>` 就不要 apply。

密码、kubeconfig、ISO、archive、qcow2、Registration URL、`registration-config.yaml`、填好的 cloud-init、私钥不要提交。

### 0.3 仓库地址不要混

注解和 `repoURL` 只写 `<host>:11443`，不要加 `/v2/`。四套地址各干一件事：

| 地址 | 谁用 | 寿命 |
|---|---|---|
| `127.0.0.1:11443` | 引导节点本机 `packtool` / `ctr push` | 引导期。KIND 拆掉即死 |
| `<KIND_REGISTRY_ADDRESS>:11443` | KIND 里 AppRelease `repoURL`、Global SeedImage `baseImage`。常见是 `172.18.0.1:11443` | 引导期。KIND 拆掉即死 |
| `<BOOTSTRAP_HOST_IP>:11443` | Global 首次 Cluster / BaremetalCluster 的 `cpaas.io/registry-address`；物理机装 OS 时拉 rootfs | 引导期。installer 之后应变为平台仓库 |
| `<PLATFORM_REGISTRY>:11443` | installer JSON `registry.domain`；成品 Global 上的 provider；Workload 全部仓库 | 永久 |

不要把 `127.0.0.1` 或 `<KIND_REGISTRY_ADDRESS>` 写进 Cluster / BaremetalCluster。物理机到不了 KIND 网桥。

installer 只改写 Cluster 的 `cpaas.io/registry-address`（同名字段如果还有 `DCSCluster` 也会改，那是 installer 自己的对象，现场不用管）。**不改写 BaremetalCluster**。交接后必须 patch，见 7.1。

### 0.4 闸门

不要凭「差不多了」往下走。

| 过了才能做 | 必须看到 |
|---|---|
| 跑 `setup.sh` | 引导节点规格够、Core Package 和解压目录在、引导节点 IP 是整场不变的静态 IP |
| 路径 A apply Registration+SeedImage | 两个 provider Ready；catalog 有 amd64 key；默认 StorageClass 能绑 20Gi PVC |
| 路径 A 挂 ISO | `status.downloadURL` 非空 |
| 路径 B apply 仅 Registration | 两个 provider Ready；catalog 有对应架构 key。**不要** apply SeedImage |
| 路径 B 建虚机 | `status.registrationURL` 非空；curl 已拿到 config；HCS 模板已是 UEFI |
| apply Pool / KCP | 三台 Inventory `Ready=True`，名称互不相同；NTP 已同步。路径 A 还要已离开 Live |
| apply `04` 导入清单 | KCP Ready、三台 Master Node Ready |
| POST installer | `04` 已 apply；JSON 已校验；密码已是 base64；curl 在引导节点打 Pod IP:8080 |
| 推镜像到平台仓库 / 拆 KIND | installer Success；第 6.3 节三项都过；Cluster **和** BaremetalCluster 已是 `<PLATFORM_REGISTRY>:11443` |
| apply 成品 Global 上的 provider | 平台仓库已有 amd64+arm64 OS 和两个 chart |
| apply Workload | 成品 catalog CM 两个 key 都对；Global 上有默认 StorageClass |

---

## 1. 进场：填值、包、网络、对时

**节点：** 引导节点。还没有 kubeconfig。

### 1.1 填对照文件

先把 `config.local.yaml` 填完，再改 YAML。现场核对盘符 WWN 时用 `hosts`，这些 WWN 不写入共享 YAML。

| 变量 | 用在哪 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | 引导节点 TLS、引导集群上的 provider、Global Registration rootfs、Global 首次 Cluster/BaremetalCluster、Global KCP etcd SAN、installer `hostIP` |
| `<KIND_REGISTRY_ADDRESS>` | 引导集群 AppRelease `repoURL` / registry、路径 A 的 Global SeedImage `baseImage` |
| `<PLATFORM_HOST>` | 成品 Global 上的 provider、installer console |
| `<PLATFORM_REGISTRY>` | installer registry、成品 Global 上的 provider、Workload 全部 registry。第 7 章才往这里推 |
| `<GLOBAL_CONTROL_PLANE_VIP>` | 引导集群上的 Bare Metal provider、Global BaremetalCluster、Global KCP etcd SAN、installer `features.ha.vip` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | Workload BaremetalCluster、Workload KCP etcd SAN |
| `<NETWORK_DEVICE>` | 所有 inventory pool、两个 BaremetalCluster |
| `<GLOBAL_POD_CIDR>` `<GLOBAL_SERVICE_CIDR>` `<GLOBAL_KUBE_OVN_JOIN_CIDR>` | Global Cluster、installer JSON |
| `<WORKLOAD_POD_CIDR>` `<WORKLOAD_SERVICE_CIDR>` `<WORKLOAD_KUBE_OVN_JOIN_CIDR>` | Workload Cluster |
| `<SSH_AUTHORIZED_KEY>` | Global KCP、Workload KCP、ARM Worker KubeadmConfigTemplate；路径 B 的 cloud-init `boot` 用户用同一把 |
| `<KUBEADM_PROVIDER_CHART_VERSION>` | 引导集群 / 成品 Global 的 Kubeadm AppRelease。从 `.ALL.` 包文件名抄 |
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | 引导集群 / 成品 Global 的 Bare Metal AppRelease。从 `.ALL.` 包文件名抄 |
| `<K25_AMD64_IMAGE_TAG>` | 第 3 章导入 amd64 archive 后才能填 |
| `<K25_ARM64_IMAGE_TAG>` | 第 7 章导入 arm64 archive 后才能填 |
| `<GLOBAL_CP_INVENTORY_1/2/3>` | Global pool、第 6.1 节导入清单。装完机后再填实际名称 |
| `<WORKLOAD_CP_INVENTORY_1/2/3>` | Workload 控制面 pool。装完机后再填实际名称 |
| `<WORKLOAD_ARM64_WORKER_INVENTORY>` | ARM Worker pool。装完机后再填实际名称 |
| `Registry` / `<BASE64_REGISTRY_PASSWORD>` | installer JSON 的仓库账号。用户名必须是 `Registry`，不是 `admin` |
| `admin@cpaas.io` / `<BASE64_PLATFORM_ADMIN_PASSWORD>` | installer JSON 的平台账号，不是仓库账号 |

CIDR 六段必须互不重叠，也不要和管理网重叠。Join 用独立网段，不要写成 Pod/Service 的子集。

### 1.2 需要准备的包

从 Customer Portal 取 **同一套 ACP 4.4.1** 产物，放到引导节点（常见 `/root`）。压缩包不进 git。

| 包 | 对照字段 | 用途 |
|---|---|---|
| Core Package | `<ACP_4_4_1_HYBRID_CORE_PACKAGE>` | `setup.sh` 起 KIND `minialauda` + 嵌入式仓库 + `cpaas-installer` |
| kubeadm provider `.ALL.` tgz | `<KUBEADM_PROVIDER_ALL_PACKAGE>` | packtool 推进引导仓库，再在第 7 章推进平台仓库 |
| Bare Metal provider `.ALL.` tgz | `<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>` | 同上 |
| K25 amd64 OCI archive | `<ACP_4_4_1_K25_AMD64_ARCHIVE>` | `kubeos-rootfs` / `kubeos-iso`，本地 source tag `baremetal` |
| K25 arm64 OCI archive | `<ACP_4_4_1_K25_ARM64_ARCHIVE>` | 第 7 章才导入。不要在引导阶段覆盖 amd64 |
| 路径 B：amd64 qcow2 | `kubeos:alaudaos-44.ku.202610100215-0.x86_64.qcow2` | 给 HCS 当系统模板，平台名 `kubeos-202610100215-x86` |
| 路径 B：arm64 qcow2 | `kubeos:alaudaos-44.ku.202610100215-0.aarch64.qcow2` | 给 HCS 当系统模板，平台名 `kubeos-202610100215-arm` |

chart 的 `targetRevision` 以包文件名为准，packtool 推完后若仓库里的版本字符串不一致，以仓库 tags 为准再改 YAML。

路径 B 把两份 qcow2 交给 HCS 上传为系统模板。启动方式必须 **UEFI**。平台「操作系统」可能显示成统信 V20，真正系统仍是这份 KubeOS。x86 模板只开 x86 虚机，ARM 模板只开 ARM 虚机。不要把已经注册过的盘再做成模板。

### 1.3 引导节点和现场网络

引导节点：传统 OS（不是 Alauda OS），静态 IP 整场不能变，必须等于后面 YAML 里的 `<BOOTSTRAP_HOST_IP>`。官方下限 x86_64 **8 核 / 16 GB / 磁盘 ≥ 300 GB**（解压目录 + `/var/cpaas` + `/var/lib/containerd`）。引导节点本身不要求 x86-64-v2；**Alauda OS 物理机**要求 x86-64-v2。ARM Worker 按现场 ARM 规格。

```bash
hostname -I
free -h
df -h / /var /var/cpaas /var/lib/containerd /root
ls -lh /root/*.tar /root/*.tgz
```

External VIP 先把转发建好，后端可以还没有：

- `<GLOBAL_CONTROL_PLANE_VIP>:6443` TCP passthrough 到三台 Global 控制面 `:6443`
- 同一 VIP 的平台入口 `80/443` 和平台仓库 `11443` 按产品拓扑转到成品 Global
- `<WORKLOAD_CONTROL_PLANE_VIP>:6443` TCP passthrough 到三台 Workload 控制面 `:6443`

路径 A：三台 Global BMC、三台 Workload BMC、一台 ARM BMC 都能挂虚拟光驱。  
路径 B：HCS 能用对应架构模板建虚机；高级选项能「用户数据注入」；CD 驱动器选不使用；软件狗关闭。模板详情若显示 Cloud-Init「不支持」，先和 HCS 确认用户数据注入仍会被虚机读到。

### 1.4 对时（装机前做）

所有即将接入的物理机和 HCS 虚机，装系统之前把时钟校好，彼此相差不超过 10 秒。路径 A 在进 Live ISO 前从 BMC/firmware 校；路径 B 在开虚机前确认 HCS 宿主机时间。漏做了对时再装机会卡 Machine，处理办法见第 10 章。

### 1.5 SSH 公钥和 installer 密码

```bash
test -f ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

把这一整行填进 `<SSH_AUTHORIZED_KEY>`。私钥不要放进仓库。

installer JSON 的两个 password 字段必须是 **base64 编码后的字符串**，不是明文：

```bash
printf 'platform admin password: ' && read -rs PLAIN && echo
printf '%s' "$PLAIN" | base64 | tr -d '\n'; echo
# 填进 <BASE64_PLATFORM_ADMIN_PASSWORD>

printf 'registry password: ' && read -rs REGPLAIN && echo
printf '%s' "$REGPLAIN" | base64 | tr -d '\n'; echo
# 填进 <BASE64_REGISTRY_PASSWORD>
# 明文自己另存，第 3 / 7 章 ctr push 要用；不要提交
```

仓库用户名固定 `Registry`。平台用户名固定 `admin@cpaas.io`。

---

## 2. Core Package 起 minialauda

**节点：** 引导节点。

把 Core Package 解到引导节点。本手册按 `/root/installer/setup.sh` 写；现场目录不同就改路径。

```bash
test -x /root/installer/setup.sh
ls /root/installer/res/amd64/kind /root/installer/res/amd64/packtool /root/installer/res/amd64/nerdctl
cd /root/installer
bash setup.sh
```

`setup.sh` 会问引导节点 IP、仓库是否设密码。填进去的 IP **必须是 `<BOOTSTRAP_HOST_IP>`**，整场不要改。脚本跑完才有 kubeconfig。

```bash
mkdir -p "${HOME}/.local/bin" ~/.kube
if ! command -v kubectl >/dev/null 2>&1; then
  export PATH="/root/installer/res/amd64:${PATH}"
  nerdctl cp minialauda-control-plane:/usr/bin/kubectl "${HOME}/.local/bin/kubectl"
  chmod 0755 "${HOME}/.local/bin/kubectl"
  export PATH="${HOME}/.local/bin:${PATH}"
fi
cp /var/cpaas/data/alauda.kubeconfig ~/.kube/config
export BOOTSTRAP_KUBECONFIG=~/.kube/config
export KUBECONFIG="$BOOTSTRAP_KUBECONFIG"
kubectl get nodes
kubectl -n cpaas-system get pods
```

成功：KIND 名 `minialauda`；本机仓库 `127.0.0.1:11443`；`cpaas-installer` 在跑。默认仓库匿名时，不要造一个空的 `global-registry-auth`。

路径 A 的 SeedImage 要 20Gi PVC。KIND 起来后立刻确认有默认 StorageClass（路径 B 不建 SeedImage，但仍建议确认，Workload 若改走 ISO 会用到）：

```bash
kubectl get sc
kubectl get sc -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{"\n"}{end}'
```

必须打出至少一个名字。空 = ISO 永远没有 `downloadURL`。

`nerdctl` 不在 PATH 时：

```bash
export PATH="/root/installer/res/amd64:${PATH}"
command -v nerdctl || command -v docker
nerdctl ps --format '{{.Names}}' | grep minialauda || docker ps --format '{{.Names}}' | grep minialauda
```

---

## 3. 推 chart 和 amd64 OS 到引导仓库

**节点：** 引导节点。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

这一步只打本机 `127.0.0.1:11443`。不要打 `<KIND_REGISTRY_ADDRESS>`，也不要打 `<PLATFORM_REGISTRY>`。平台仓库此时还不存在。

### 3.1 packtool 推两个 provider chart

```bash
PACKTOOL=/root/installer/res/amd64/packtool
test -x "$PACKTOOL"
$PACKTOOL pack push -h
$PACKTOOL pack push -r "127.0.0.1:11443" -c /root/<KUBEADM_PROVIDER_ALL_PACKAGE>
$PACKTOOL pack push -r "127.0.0.1:11443" -c /root/<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>
```

先看 help，不要假设认证开关叫 `-u`。引导仓库默认匿名。`setup.sh` 给仓库设了账号时，按 help 里的认证参数再推。

### 3.2 导入 amd64 KubeOS，读最终 tag，推进引导仓库

`/var/lib/containerd` 至少要有归档那么大的空闲。不要先 `ctr images rm`：交接后还要从同一个 namespace 再推到平台仓库。

```bash
command -v ctr
ctr namespaces ls || true
ctr namespaces create bare-metal-os 2>/dev/null || true
ctr -n bare-metal-os images import --all-platforms --no-unpack /root/<ACP_4_4_1_K25_AMD64_ARCHIVE>
ctr -n bare-metal-os images ls -q
```

必须看到本地 source：

```text
kubeos-rootfs:baremetal
kubeos-iso:baremetal
```

最终 tag 以这次 `images ls` 和现场 archive 说明为准，填进 `config.local.yaml` 的 `<K25_AMD64_IMAGE_TAG>`，再替换 YAML。不要把 `baremetal` 写进 YAML。

```bash
export OS_TAG=<K25_AMD64_IMAGE_TAG>
for repo in kubeos-rootfs kubeos-iso; do
  ctr -n bare-metal-os images tag \
    "${repo}:baremetal" "${repo}:${OS_TAG}"
  ctr -n bare-metal-os images push --plain-http \
    "127.0.0.1:11443/tkestack/${repo}:${OS_TAG}" \
    "${repo}:${OS_TAG}"
  curl -s "http://127.0.0.1:11443/v2/tkestack/${repo}/tags/list"
done
```

KIND 里通过 `<KIND_REGISTRY_ADDRESS>:11443` 读到的就是刚才推进 `127.0.0.1:11443` 的同一份。物理机装 OS 时走 `<BOOTSTRAP_HOST_IP>:11443`。

arm64 archive **这一步不要导入**，会覆盖本地 `*:baremetal` source tag。

---

# 第一部分：创建 Global 集群

本章全部在引导节点执行，kubeconfig 指向 KIND `minialauda`：

```bash
export KUBECONFIG="$BOOTSTRAP_KUBECONFIG"
```

---

## 4. Global：在引导集群上装 provider

```bash
kubectl get nodes
kubectl get storageclass
kubectl -n cpaas-system get pods
```

KIND 名必须是 `minialauda`，并且有默认 StorageClass。

### 4.1 `global/bootstrap/01-dex-tls-bootstrap.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | Certificate `spec.commonName`、`spec.ipAddresses` |

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/01-dex-tls-bootstrap.yaml"
kubectl -n cpaas-system get certificate dex-tls-bootstrap
kubectl -n cpaas-system get secret dex.tls
```

`Ready=True` 后再装 Bare Metal provider。这个 Secret 不要导入成品 Global。

### 4.2 `global/bootstrap/02-kubeadm-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<KUBEADM_PROVIDER_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<KIND_REGISTRY_ADDRESS>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<KIND_REGISTRY_ADDRESS>:11443` |
| `<BOOTSTRAP_HOST_IP>` | `values.global.host`、`values.global.platformUrl`（`https://<BOOTSTRAP_HOST_IP>`） |

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/02-kubeadm-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease cluster-api-provider-kubeadm
```

### 4.3 `global/bootstrap/03-baremetal-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<KIND_REGISTRY_ADDRESS>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<KIND_REGISTRY_ADDRESS>:11443` |
| `<BOOTSTRAP_HOST_IP>` | `values.global.host`、`values.global.platformUrl`（`https://<BOOTSTRAP_HOST_IP>`）、`values.elemental.server.url`（`https://<BOOTSTRAP_HOST_IP>:12443`） |
| `<K25_AMD64_IMAGE_TAG>` | `values.provider.imageCatalog.images.v1.35.6-acp.9_amd64_alaudaos.k.25.tag` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `values.provider.handoffHook.controlPlaneVIP` |

这一步 catalog **只填 amd64**。不要在引导阶段加 arm64 catalog。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/03-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

catalog 必须有：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25: <KIND_REGISTRY_ADDRESS>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
```

两个 AppRelease Ready、catalog 有 amd64 key 后再继续。

---

## 5. Global：接入三台控制面，创建 Cluster

三台 Global 控制面必须同一条路径。A 和 B **不要**对同一台机器一起做，也不要 apply 两份 `01`（Registration 同名会冲突）。

### 5.1A 路径 A：`global/control-plane/01-registration-seedimage-amd64.yaml`

裸金属、能挂 ISO 时用这份。会同时创建 `global-registration` 和 SeedImage。

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
kubectl -n cpaas-system get pvc
ISO_URL=""
for i in $(seq 1 60); do
  ISO_URL=$(kubectl -n cpaas-system get seedimage global-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
echo "$ISO_URL"
```

PVC Pending = 没有默认 StorageClass，不要干等。闸门是 `status.downloadURL` 非空，不是某个固定的 condition 名。

在 **引导节点**上下 ISO（URL 通常只在集群网络内可访问）：

```bash
curl -fL --retry 3 -o /root/global-registration.iso "$ISO_URL"
ls -lh /root/global-registration.iso
```

把 ISO 拷到能挂这三台 BMC 虚拟光驱的那台机器。这个 ISO 只给三台 Global 控制面，不要给 Workload 用。`cleanupAfterMinutes: 120` 过后构建产物会被清。

Registration URL 只在引导节点终端看一眼，不要写进仓库、工单、聊天：

```bash
kubectl -n cpaas-system get machineregistration global-registration \
  -o jsonpath='{.status.registrationURL}{"\n"}'
```

### 5.1B 路径 B：`global/control-plane/01-registration-amd64.yaml`

HCS 虚机、不能挂 ISO 时用这份。只创建 `global-registration`，**不要** apply SeedImage 那份。

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `system-uri`，写成 `<BOOTSTRAP_HOST_IP>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>` |
| `<K25_AMD64_IMAGE_TAG>` | `system-uri` 的 tag |

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/01-registration-amd64.yaml"
URL=""
for i in $(seq 1 30); do
  URL=$(kubectl -n cpaas-system get machineregistration global-registration \
    -o jsonpath='{.status.registrationURL}' 2>/dev/null || true)
  [ -n "$URL" ] && break
  echo "waiting registrationURL ($i/30)..."
  sleep 10
done
test -n "$URL"
curl -fsSk "$URL" > /root/registration-config.yaml
chmod 600 /root/registration-config.yaml
```

`registrationURL` 和这份 yaml 都是凭证，不要提交。x86 虚机只用 amd64 这条 URL。

复制样例，把 curl 结果原样缩进贴进 `write_files` 里 `/etc/elemental/registration/config.yaml` 的 `content: |`，公钥换成 `<SSH_AUTHORIZED_KEY>`：

```bash
cp "$PACK_DIR/config/cloud-init-user-data.example.yaml" /root/cloud-init-user-data.yaml
```

粘贴后先在引导节点检查缩进，再贴到 HCS。这条命令需要 Python 的 PyYAML（`python3 -c 'import yaml'` 报错就先装 `python3-yaml` / `PyYAML`）：

```bash
python3 -c 'import yaml; yaml.safe_load(open("/root/cloud-init-user-data.yaml"))'
```

### 5.2A 路径 A：三台 Global 物理机装系统

**节点：** 每台 Global 物理机 BMC / Live ISO。不用 kubectl。

进 Live 前确认这三台已经对时（见 1.4）。BMC 挂上 5.1A 的 ISO，从光驱启动。Live 控制台一般是 root、无密码（以现场 ISO 提示为准）。

无 DHCP 时先配静态 IP，再做软链。地址按 **这台机器** 改：

```bash
ip -br link
ip -br addr
nmcli con add type ethernet ifname <NETWORK_DEVICE> con-name mgmt \
  ipv4.method manual \
  ipv4.addresses <这台机器管理地址>/<前缀> \
  ipv4.gateway <网关> \
  ipv4.dns <DNS> \
  ipv6.method ignore \
  connection.autoconnect yes
nmcli con up mgmt
ping -c 3 <BOOTSTRAP_HOST_IP>
nc -vz <BOOTSTRAP_HOST_IP> 12443 || curl -skI "https://<BOOTSTRAP_HOST_IP>:12443" | head
nc -vz <BOOTSTRAP_HOST_IP> 11443 || curl -sI "http://<BOOTSTRAP_HOST_IP>:11443/v2/" | head
```

有 DHCP 且能 ping 通引导节点就跳过配网。

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

不要链到 ISO、loop 或数据盘。软链完成后不要再手搓 `elemental-register`。看进度：

```bash
journalctl -f | grep -Ei 'elemental|register|install|cloud-init' || true
```

装完：弹出虚拟光驱，BMC 改成硬盘优先，确认下次进已安装 OS。`/` 不能还是 Live。已离开 Live 的机器不要再回 Live 重建链接。

### 5.2B 路径 B：三台 Global HCS 虚机

**节点：** HCS 控制台。模板用 `kubeos-202610100215-x86`，启动 **UEFI**。每台虚机从干净模板新建，不要克隆已注册的盘。

高级选项：

- 软件狗：关闭
- CD 驱动器：不使用
- 用户数据注入：文本，把 `/root/cloud-init-user-data.yaml` **全文**贴进去

三台都用 **同一份** Global amd64 的 cloud-init（同一条 `global-registration` URL）。开机后 cloud-init 跑 `elemental-register --install --no-toolkit`。网络要能访问引导节点 `:12443` 和 `:11443`。

系统盘默认 `/dev/vda`。qcow2 约 1.8 GB，HCS 盘常见 100 GB，空间不会自动变成根分区。SSH 进去后若 `df -h /` 仍很小：

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
findmnt -n -o SOURCE,FSTYPE,SIZE,AVAIL /
```

确认 `/` 在 `/dev/vda4`、ext4、盘尾还有空闲，再扩（不要动 vda1 EFI）：

```bash
sfdisk --dump /dev/vda > /root/vda.sfdisk.bak
echo ", +" | sfdisk --no-reread -N 4 /dev/vda
partx -u /dev/vda
resize2fs /dev/vda4
```

已经顶到盘尾就跳过。分区号或文件系统不是 vda4/ext4，先改命令再跑。扩盘可以在等 Inventory 之前做。

在这台虚机上看注册：

```bash
sudo less /var/log/preinst-register.log
sudo journalctl -fu elemental-system-agent
```

### 5.2C 两条路径汇合：抄 Inventory 名

回到引导节点：

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

三个 inventory `Ready=True` 后，把实际名称填进 `config.local.yaml` 的：

```text
<GLOBAL_CP_INVENTORY_1>
<GLOBAL_CP_INVENTORY_2>
<GLOBAL_CP_INVENTORY_3>
```

三个名称必须互不相同。路径 A 还要已从系统盘启动。此时 **不要** apply `04`。

### 5.3 `global/control-plane/02-control-plane-pool.yaml`

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

### 5.4 `global/control-plane/03-baremetal-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `metadata.annotations.cpaas.io/registry-address`，首次创建写成 `<BOOTSTRAP_HOST_IP>:11443` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `spec.controlPlaneLoadBalancer.host` |
| `<NETWORK_DEVICE>` | `spec.networkDevice` |

首次 apply 不要改成平台仓库。`type: External`、`port: 6443` 不要改。installer 成功后如果这个对象还停在引导仓库，只按 7.1 patch，不要重新 apply 整个文件。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/03-baremetal-cluster.yaml"
```

### 5.5 `global/control-plane/04-control-plane-machine-template.yaml`

无变量。确认 `machineInventoryPoolRef.name` 是 `global-control-plane-pool`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/04-control-plane-machine-template.yaml"
```

### 5.6 `global/control-plane/05-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `metadata.annotations.cpaas.io/registry-address`，首次创建写成 `<BOOTSTRAP_HOST_IP>:11443` |
| `<GLOBAL_KUBE_OVN_JOIN_CIDR>` | `metadata.annotations.cpaas.io/kube-ovn-join-cidr` |
| `<GLOBAL_POD_CIDR>` | `spec.clusterNetwork.pods.cidrBlocks[0]` |
| `<GLOBAL_SERVICE_CIDR>` | `spec.clusterNetwork.services.cidrBlocks[0]` |

Kubernetes 版本保持 `v1.35.6-acp.9`，Kube-OVN 保持 `v4.4.12`。installer 会改这个 Cluster 的 registry annotation，成功后不要重新 apply 这个文件。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/05-cluster.yaml"
```

### 5.7 `global/control-plane/06-kubeadm-control-plane.yaml`

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

### 5.8 抽出 global-kubeconfig（还不能拆 KIND）

```bash
kubectl -n cpaas-system get secret global-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > "$PACK_DIR/global-kubeconfig"
chmod 600 "$PACK_DIR/global-kubeconfig"
export GLOBAL_KUBECONFIG="$PACK_DIR/global-kubeconfig"
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" get nodes -o wide
```

期待：三台 Master Node Ready；KCP `Ready=True`；Cluster `Phase: Provisioned`。此时还不是拆 KIND 的时候，也还没 POST installer。

---

## 6. Global：交接和 installer

**节点：** 引导节点。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

### 6.1 `global/bootstrap/04-dcs-import-extra-resources.yaml`

从 **已经绑上 Inventory 的** BaremetalMachine 抄名字，不要用尚未被 Machine 认领的列表：

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

三个名字必须和 Pool 一致，且互不相同。`global-registration` 已写死，不要改。不要往这个文件加 SeedImage、token、plan、bootstrap TLS。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/04-dcs-import-extra-resources.yaml"
kubectl -n cpaas-system get cm dcs-import-extra-resources -o yaml
```

缺这个 ConfigMap 时，交接目标列表为空，成品 Global 没有 BaremetalMachine / Inventory / Registration。文件名里的 `dcs-` 是 installer 接口要求的，不要改名。

### 6.2 `global/bootstrap/05-installer-config.example.json`

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
| `<PLATFORM_HOST>` | `console.host[0]`、`console.globalHost`。不要带 `https://`，不要带端口 |
| `<GLOBAL_POD_CIDR>` | `cluster.clusterCIDR` |
| `<GLOBAL_SERVICE_CIDR>` | `cluster.serviceCIDR` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `cluster.features.ha.vip` |
| `<BOOTSTRAP_HOST_IP>` | `hostIP` |

`deployMode` 保持 `normal`。`features.ha` 是 installer 收 API VIP 的字段。不要设 `registry.externalAddress`。

```bash
python3 -m json.tool /root/installer-config.json >/dev/null
```

`basic.password` / `registry.password` 看起来应是字母数字和 `=`，不是刚敲的明文。

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

另一个终端提交。必须在引导节点打 installer Pod IP 的 `8080`，不要打平台 VIP。这个 Pod IP 只在引导集群网络里通，别的机器 curl 不到：

```bash
curl -sS -m 5 "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress" | head
curl -fS -X POST \
  "http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs" \
  -H 'Content-Type: application/json' \
  --data-binary @/root/installer-config.json
```

监控（通常 30–60 分钟）：

```bash
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
tail -f /var/cpaas/data/installer.log
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" get nodes
```

installer Success 不等于可以拆引导集群。

### 6.3 交接完成的三项检查

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system \
  get configmap baremetal-system-agent-handoff -o jsonpath='{.data.ready}{"\n"}'
# 必须是 true。不要手工 patch。

kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system \
  get machineinventories.elemental.cattle.io \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.plan.state}{"\n"}{end}'
# 每台导入的 Global 主机 status.plan.state = Applied

kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system \
  get rolebinding baremetal-global-system-agent-handoff-bridge
# 必须 NotFound
```

三项都过后再做第 7 章。KIND 先留着，引导节点还要用来 `ctr push`。

---

## 7. Global：推到平台仓库、成品 provider、catalog CM

**节点：** 引导节点。OS 推送仍在引导节点做。从此查看成品集群改用 `$GLOBAL_KUBECONFIG`。

引导仓库里的 ISO / catalog / chart **不会**跟着 installer 过去。不推到平台仓库，Workload 会 `MANIFEST_UNKNOWN`。

### 7.1 确认并 patch registry 注解

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system \
  get cluster,baremetalcluster global -o \
  custom-columns='KIND:.kind,REG:.metadata.annotations.cpaas\.io/registry-address'
```

期待两边都是 `<PLATFORM_REGISTRY>:11443`。Cluster 仍是 `<BOOTSTRAP_HOST_IP>:11443` 时，先确认 installer Success。BaremetalCluster 官方不改写，按下面 patch：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system annotate baremetalcluster global \
  cpaas.io/registry-address=<PLATFORM_REGISTRY>:11443 --overwrite
```

不要出现 `127.0.0.1` / `<KIND_REGISTRY_ADDRESS>` / 残留的 `<BOOTSTRAP_HOST_IP>:11443`。不要重新 apply `03` 或 `05`。

### 7.2 推 amd64 OS 到平台仓库

按现场策略配置平台仓库证书信任，不要关 TLS 校验。用户名是 `Registry`，密码是 1.5 生成的明文（不是 base64）。

```bash
export OS_TAG=<K25_AMD64_IMAGE_TAG>
export REGISTRY=<PLATFORM_REGISTRY>:11443
ctr -n bare-metal-os images ls -q | grep "$OS_TAG"
for repo in kubeos-rootfs kubeos-iso; do
  ctr -n bare-metal-os images push \
    -u "Registry:<REGISTRY_PASSWORD>" \
    "$REGISTRY/tkestack/${repo}:${OS_TAG}" \
    "${repo}:${OS_TAG}"
done
```

### 7.3 导入 arm64 archive，推到平台仓库

amd64 最终 tag 已经打好后再导入 arm64，避免覆盖 `*:baremetal` 时把还没 tag 的 amd64 弄丢。

```bash
ctr -n bare-metal-os images import --all-platforms --no-unpack /root/<ACP_4_4_1_K25_ARM64_ARCHIVE>
ctr -n bare-metal-os images ls -q
```

把读到的最终 tag 填进 `<K25_ARM64_IMAGE_TAG>`。两个最终 tag 不能互相覆盖。

```bash
export OS_TAG=<K25_ARM64_IMAGE_TAG>
export REGISTRY=<PLATFORM_REGISTRY>:11443
for repo in kubeos-rootfs kubeos-iso; do
  ctr -n bare-metal-os images tag \
    "${repo}:baremetal" "${repo}:${OS_TAG}"
  ctr -n bare-metal-os images push \
    -u "Registry:<REGISTRY_PASSWORD>" \
    "$REGISTRY/tkestack/${repo}:${OS_TAG}" \
    "${repo}:${OS_TAG}"
done
```

平台仓库此时必须同时有：

```text
tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
tkestack/kubeos-iso:<K25_AMD64_IMAGE_TAG>
tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>
tkestack/kubeos-iso:<K25_ARM64_IMAGE_TAG>
```

### 7.4 packtool 把两个 chart 推到平台仓库

```bash
PACKTOOL=/root/installer/res/amd64/packtool
$PACKTOOL pack push -h
$PACKTOOL pack push -r "<PLATFORM_REGISTRY>:11443" \
  -c /root/<KUBEADM_PROVIDER_ALL_PACKAGE>
$PACKTOOL pack push -r "<PLATFORM_REGISTRY>:11443" \
  -c /root/<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>
```

认证参数按 help。仓库账号是 `Registry`。

### 7.5 `global/control-plane/07-finished-kubeadm-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<KUBEADM_PROVIDER_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<PLATFORM_REGISTRY>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<PLATFORM_HOST>` | `values.global.host`、`values.global.platformUrl`（`https://<PLATFORM_HOST>`） |

这个文件必须 apply 到成品 Global，不要 apply 到 `minialauda`。

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl apply -f "$PACK_DIR/global/control-plane/07-finished-kubeadm-provider-apprelease.yaml"
```

### 7.6 `global/control-plane/08-finished-baremetal-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<PLATFORM_REGISTRY>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<PLATFORM_HOST>` | `values.global.host`、`values.global.platformUrl`（`https://<PLATFORM_HOST>`）、`values.elemental.server.url`（`https://<PLATFORM_HOST>`，不要加 `:12443`） |
| `<K25_AMD64_IMAGE_TAG>` | amd64 catalog tag |
| `<K25_ARM64_IMAGE_TAG>` | arm64 catalog tag |

这里必须同时有两个 catalog key，tag 不能交叉。

先确认成品 `dex.tls` 的 key。installer 自签证书通常只有 `tls.crt` / `tls.key`，本文件 `caCertSecretKey` 已写成 `tls.crt`。若 Secret 里是 `ca.crt`，把 YAML 改回 `ca.crt` 再 apply。

```bash
kubectl -n cpaas-system get secret dex.tls -o jsonpath='{.data}'; echo
kubectl apply -f "$PACK_DIR/global/control-plane/08-finished-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
```

### 7.7 检查 ConfigMap `cpaas-system/elemental-image-catalog`

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

CM 对象不存在时，先查 `08` 是否 Ready。两个 key 都对了，再确认 Global 上有默认 StorageClass，然后才可以拆 KIND、创建 Workload。

拆 KIND 之前再扫一遍 Cluster / BaremetalCluster 注解必须是 `<PLATFORM_REGISTRY>:11443`。拆完 `127.0.0.1` / `<KIND_REGISTRY_ADDRESS>` 会失效。

---

# 第二部分：创建 Workload 集群

本章全部使用 `$GLOBAL_KUBECONFIG`。KIND 已不再使用。

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl get nodes
kubectl get sc
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

---

## 8. Workload：amd64 控制面

和 Global 一样，三台 Workload 控制面只选路径 A 或路径 B。不要复用 Global 的 ISO、Registration URL 或 cloud-init。

### 8.1A 路径 A：`workload/control-plane/01-registration-seedimage-amd64.yaml`

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
ISO_URL=""
for i in $(seq 1 60); do
  ISO_URL=$(kubectl -n cpaas-system get seedimage hcs-workloadcluster-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
```

这个 ISO 只给三台 Workload amd64 控制面。不要复用 Global ISO。

### 8.1B 路径 B：`workload/control-plane/01-registration-amd64.yaml`

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/01-registration-amd64.yaml"
URL=""
for i in $(seq 1 30); do
  URL=$(kubectl -n cpaas-system get machineregistration hcs-workloadcluster-registration \
    -o jsonpath='{.status.registrationURL}' 2>/dev/null || true)
  [ -n "$URL" ] && break
  sleep 10
done
test -n "$URL"
curl -fsSk "$URL" > /root/registration-config-workload-cp.yaml
```

cloud-init 样例和 5.1B 相同，换成这一份 curl 结果。HCS 模板仍是 `kubeos-202610100215-x86`。虚机网络走 `<PLATFORM_HOST>:443` 和 `<PLATFORM_REGISTRY>:11443`，不是引导节点 `:12443`。

### 8.2A 路径 A：三台 Workload 控制面物理机

**节点：** 每台 Workload amd64 控制面 BMC / Live ISO。

进 Live 前对时。WWN 对照：

```text
<WORKLOAD_CP_1_OS_DISK_WWN>
<WORKLOAD_CP_2_OS_DISK_WWN>
<WORKLOAD_CP_3_OS_DISK_WWN>
```

symlink、无 DHCP 配网、弹出光驱、改硬盘优先，流程与 5.2A 相同。网络要能访问 `<PLATFORM_HOST>` 的 443 和 `<PLATFORM_REGISTRY>:11443`，不是引导节点 `:12443`。

### 8.2B 路径 B：三台 Workload 控制面虚机

流程与 5.2B 相同：x86 模板、UEFI、用户数据注入、可选扩 `/dev/vda4`。三台共用 Workload amd64 这份 cloud-init。

### 8.2C 汇合：抄 Inventory 名

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

### 8.3 `workload/control-plane/02-control-plane-pool.yaml`

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

### 8.4 `workload/control-plane/03-baremetal-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `metadata.annotations.cpaas.io/registry-address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | `spec.controlPlaneLoadBalancer.host` |
| `<NETWORK_DEVICE>` | `spec.networkDevice` |

不要写成引导仓库或 `127.0.0.1`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/03-baremetal-cluster.yaml"
```

### 8.5 `workload/control-plane/04-control-plane-machine-template.yaml`

无变量。确认 `machineInventoryPoolRef.name` 是 `hcs-workloadcluster-control-plane-pool`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/04-control-plane-machine-template.yaml"
```

### 8.6 `workload/control-plane/05-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `metadata.annotations.cpaas.io/registry-address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<WORKLOAD_KUBE_OVN_JOIN_CIDR>` | `metadata.annotations.cpaas.io/kube-ovn-join-cidr` |
| `<WORKLOAD_POD_CIDR>` | `spec.clusterNetwork.pods.cidrBlocks[0]` |
| `<WORKLOAD_SERVICE_CIDR>` | `spec.clusterNetwork.services.cidrBlocks[0]` |

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/05-cluster.yaml"
```

### 8.7 `workload/control-plane/06-kubeadm-control-plane.yaml`

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

## 9. Workload：arm64 Worker

这台 ARM Worker 也可以走路径 A 或路径 B。不要用 amd64 的 ISO、qcow2、Registration URL 或 cloud-init。

### 9.1A 路径 A：`workload/worker/01-registration-seedimage-arm64.yaml`

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
ISO_URL=""
for i in $(seq 1 60); do
  ISO_URL=$(kubectl -n cpaas-system get seedimage \
    hcs-workloadcluster-arm64-worker-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
```

### 9.1B 路径 B：`workload/worker/01-registration-arm64.yaml`

```bash
kubectl apply -f "$PACK_DIR/workload/worker/01-registration-arm64.yaml"
URL=""
for i in $(seq 1 30); do
  URL=$(kubectl -n cpaas-system get machineregistration \
    hcs-workloadcluster-arm64-worker-registration \
    -o jsonpath='{.status.registrationURL}' 2>/dev/null || true)
  [ -n "$URL" ] && break
  sleep 10
done
test -n "$URL"
curl -fsSk "$URL" > /root/registration-config-workload-arm.yaml
```

cloud-init 样例和 5.1B 相同，换成这一份 curl 结果。HCS 模板用 `kubeos-202610100215-arm`，启动仍是 UEFI。

### 9.2A 路径 A：ARM 物理机

**节点：** 那一台 ARM Worker BMC / Live ISO。

进 Live 前对时。WWN 对照 `<WORKLOAD_ARM64_WORKER_1_OS_DISK_WWN>`。symlink 和配网流程与 5.2A 相同。

### 9.2B 路径 B：ARM 虚机

流程与 5.2B 相同，模板换成 ARM 那份。系统盘仍默认 `/dev/vda`，扩盘命令相同。

### 9.2C 汇合

装完后把实际 inventory 名称填进 `<WORKLOAD_ARM64_WORKER_INVENTORY>`。

### 9.3 `workload/worker/02-worker-pool-arm64.yaml`

| 变量 | 字段 |
|---|---|
| `<WORKLOAD_ARM64_WORKER_INVENTORY>` | `spec.machineInventories[0].name` |
| `<NETWORK_DEVICE>` | `networkDevice` |

hostname 保持 `hcs-workloadcluster-arm64-worker-1`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/02-worker-pool-arm64.yaml"
```

### 9.4 `workload/worker/03-worker-machine-template-arm64.yaml`

无变量。确认 `machineInventoryPoolRef.name` 是 `hcs-workloadcluster-arm64-worker-pool`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/03-worker-machine-template-arm64.yaml"
```

### 9.5 `workload/worker/04-worker-kubeadm-config-template-arm64.yaml`

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

### 9.6 `workload/worker/05-worker-machine-deployment-arm64.yaml`

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

## 10. 运维

先看日志，再删单个对象。删 `Cluster` 是最后手段，且只能删 Workload，不能删 `Cluster/global`。

### 10.1 时间同步

漏做了对时：先把相关物理机时间同步好，再删卡住的 `Machine`，看控制器会不会重建。

```bash
date -u
timedatectl show -p NTPSynchronized --value
chronyc tracking
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system get machine -o wide
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system delete machine <FAILED_MACHINE_NAME>
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system get machine,baremetalmachine -o wide \
  -l cluster.x-k8s.io/cluster-name=<CLUSTER_NAME>
```

### 10.2 引导节点上看 installer

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

### 10.3 触发重新安装

资源已经 apply 完但卡住时，先删失败的 `Machine`：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system get machine,baremetalmachine -o wide
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system delete machine <FAILED_MACHINE_NAME>
```

旧对象被清掉、新对象被拉起，说明上一轮已经清完并开始重装。

`ImageCatalogMiss`：先按 7.7 补 catalog CM，再只删失败的 `BaremetalMachine`。不要删 Cluster。

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

### 10.4 路径 B：虚机注册日志和扩盘

在这台虚机上看：

```bash
sudo less /var/log/cloud-init-output.log
sudo less /var/log/preinst-register.log
sudo journalctl -fu elemental-system-agent
```

根分区没自动变大时，先 `lsblk` / `findmnt` 确认 `/` 在 `/dev/vda4` 且是 ext4，再按 5.2B 扩盘。不要扩 EFI 分区。

### 10.5 登录机器看 cloud-init / elemental-agent

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

### 10.6 kubelet / containerd

```bash
sudo journalctl -fu kubelet
sudo journalctl -fu containerd
sudo crictl ps -a
sudo systemctl status kubelet containerd
```

节点上如果已经出现 `"staticPodPath": null`，先改对应 YAML，再按 10.3 重建那台 Machine。路径 B 重建时从干净 qcow2 模板再开一台，贴同一角色的 cloud-init，不要克隆已注册磁盘。

### 10.7 引导节点上看 cluster-transformer

installer 还没成功时，看引导集群：

```bash
kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cluster-transformer \
  --all-containers=true --prefix=true
```

installer 成功之后，看成品 Global：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cluster-transformer \
  --all-containers=true --prefix=true
```

installer 看 Global 首次安装；transformer 看后续插件和集群变更。
