# HCS Bare Metal ACP 4.4.1 交付手册

固定拓扑、单站点、YAML-only。对照文件只有 `config/config.example.yaml`。选定一种主机形态后只走对应场景，不要混用两套 Registration。

| 现场 | 阅读顺序 |
|---|---|
| 共用准备 | 第 0–4 章 |
| HCS 虚机 | 第 5 章 → 第 6–7 章 → 第 8–9 章 |
| 同网段物理机 | 第 10 章 → 第 6–7 章 → 第 11–12 章 |
| 卡住 | 第 13 章 |

Inventory `Ready=True` 之后，Pool / Cluster / KCP / MachineDeployment 的 YAML 两场景相同（第 5.4–5.9、8.4–8.8、9.3–9.6 节）。差别只在主机如何注册：

- HCS：`01-registration-*.yaml`（不要 SeedImage）。闸门是 `status.registrationURL`。
- 物理机：`01-registration-seedimage-*.yaml`。闸门是 SeedImage `status.downloadURL`。

两份 Registration **同名**，禁止同时 apply。裸金属现场不要因为 5.4 写在场景一标题下而跳过：那几节是共用 apply 步骤。

## 0. 范围与约定

| 项 | 值 |
|---|---|
| Global | 3 台 amd64 控制面，无 Global Worker |
| Workload | 3 台 amd64 控制面 + 1 台 arm64 Worker |
| 集群名 | Global 必须为 `global`。Workload 用 `<WORKLOAD_CLUSTER_NAME>`，对照 `topology.workload_cluster_name`；本场示例 `workloadcluster`。禁止写成 `global`，同一场只许一个名字 |
| 滚动 | 控制面与 Worker 的 `maxSurge` 一律为 `0` |
| 负载均衡 | External VIP；不要配置 VRID / Internal LB |

| 术语 | 含义 |
|---|---|
| 引导节点 | 解 Core Package、执行 `setup.sh`、拉起 KIND 的传统 OS 主机。IP 即 `<BOOTSTRAP_HOST_IP>` |
| KIND / `minialauda` | `setup.sh` 在引导节点上创建的引导集群，用于创建 Global。installer 交接完成后拆除 |
| 成品 Global | 三台控制面就绪且 installer Success 之后的管理集群。此后 Workload 全部 apply 到这里 |
| 引导仓库 | KIND 嵌入式仓库。本机 `127.0.0.1:11443`，KIND 内常见 `<KIND_REGISTRY_ADDRESS>:11443` |
| 平台仓库 | installer 起来之后的 `<PLATFORM_REGISTRY>:11443`，永久使用 |

版本、catalog、账号以 `config/config.example.yaml` 为准。现场核对这些固定值：

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
catalog      v1.35.6-acp.9_amd64_alaudaos.k.25
             v1.35.6-acp.9_arm64_alaudaos.k.25
平台账号     admin
仓库账号     registry
```

`baremetal` 只是 archive 导入后的本地 source tag，禁止写入 YAML。最终 tag 以 `ctr -n bare-metal-os images ls -q` 为准：`<K25_AMD64_IMAGE_TAG>` / `<K25_ARM64_IMAGE_TAG>`。本场 amd64 成功值为 `alaudaos-44.ku.202610100215-0.x86_64`，下一场仍以现场 `images ls` 为准。不要猜测 ARM 后缀。

kubelet patch 必须包含 `imagePullCredentialsVerificationPolicy: NeverVerify`，且不要出现 `staticPodPath`（包括 `null`）。

### 0.1 执行位置与 kubeconfig

第 2 章 `setup.sh` 完成后才有引导集群 kubeconfig。Global KCP Ready 后才能从 Secret 抽出 `global-kubeconfig`。进场时先只设置包目录：

```bash
export PACK_DIR=/path/to/baremetal-hcs
```

| 阶段 | 执行节点 | kubeconfig | 目标 |
|---|---|---|---|
| 第 2–4 章，以及 POST installer 之前 | 引导节点 | `$BOOTSTRAP_KUBECONFIG`（第 2 章生成） | KIND `minialauda` |
| HCS 装 OS | HCS 控制台 | 无 | 该虚机；cloud-init 在虚机内执行 |
| 裸金属装 OS | 对应物理机 BMC / Live ISO | 无 | 该物理机 |
| installer Success 且已抽出 kubeconfig | 引导节点 | `$GLOBAL_KUBECONFIG` | 成品 Global |
| 第 13 章 | 见各命令 | 见各命令 | 见各命令 |

### 0.2 对照文件

唯一对照文件：`config/config.example.yaml`。

```bash
cp "$PACK_DIR/config/config.example.yaml" "$PACK_DIR/config/config.local.yaml"
```

每一步列出的变量，从 `config.local.yaml` 取现场值，手工替换到该步 YAML / JSON。改对照文件不会自动改 manifests。apply 前：

```bash
grep -nE '<[^>]+>' 待 apply 的目标文件
```

文件中仍残留 `<...>` 则不要 apply。

密码、kubeconfig、ISO、archive、qcow2、Registration URL、`registration-config.yaml`、填好的 cloud-init、私钥禁止提交。

### 0.3 四套仓库地址

注解和 `repoURL` 只写 `<host>:11443`，不要加 `/v2/`。

| 地址 | 用途 | 寿命 |
|---|---|---|
| `127.0.0.1:11443` | 引导节点本机 `packtool` / `ctr push` | 引导期。KIND 拆除即失效 |
| `<KIND_REGISTRY_ADDRESS>:11443` | KIND 内 AppRelease `repoURL`、物理机场景 Global SeedImage `baseImage`。常见 `172.18.0.1:11443` | 引导期。KIND 拆除即失效 |
| `<BOOTSTRAP_HOST_IP>:11443` | Global 首次 Cluster / BaremetalCluster 的 `cpaas.io/registry-address`；首次装 OS 拉 rootfs | 引导期。installer 之后必须改为平台仓库 |
| `<PLATFORM_REGISTRY>:11443` | installer JSON `registry.domain`；成品 Global provider；Workload 全部仓库 | 永久 |

不要把 `127.0.0.1` 或 `<KIND_REGISTRY_ADDRESS>` 写入 Cluster / BaremetalCluster。物理机和 HCS 虚机都到不了 KIND 网桥。

installer 只改写 Cluster 的 `cpaas.io/registry-address`（同名字段如果还有 `DCSCluster` 也会改，那是 installer 自有对象）。**不改写 BaremetalCluster**。交接后必须按第 7.1 节 patch。

### 0.4 闸门

未满足闸门条件不得进入下一步。

共用：

| 过了才能做 | 必须看到 |
|---|---|
| 执行 `setup.sh` | 引导节点规格足够；Core Package 与解压目录存在；引导节点 IP 为整场不变的静态 IP |
| apply `04` 导入清单 | KCP Ready；三台 Master Node Ready |
| POST installer | `04` 已 apply；JSON 已校验；密码已是 base64；在引导节点 curl Pod IP:8080 |
| 推镜像到平台仓库 / 拆 KIND | installer Success；交接三项检查通过；Cluster **和** BaremetalCluster 均为 `<PLATFORM_REGISTRY>:11443` |
| apply 成品 Global 上的 provider | 平台仓库已有 amd64+arm64 OS 和两个 chart |
| apply Workload | 成品 catalog ConfigMap 两个 key 均正确 |

HCS 另加：

| 过了才能做 | 必须看到 |
|---|---|
| apply 仅 Registration | 两个 provider Ready；catalog 有对应架构 key。**不要** apply SeedImage |
| 创建虚机 | `status.registrationURL` 非空；已 curl 到 config；HCS 模板为 UEFI |
| apply Pool / KCP | 三台 Inventory `Ready=True` 且名称互不相同；NTP 已同步 |

裸金属另加：

| 过了才能做 | 必须看到 |
|---|---|
| apply Registration+SeedImage | 两个 provider Ready；catalog 有对应架构 key；默认 StorageClass 能绑定 20Gi PVC |
| 挂 ISO | `status.downloadURL` 非空 |
| apply Pool / KCP | 三台 Inventory `Ready=True` 且名称互不相同；NTP 已同步；已离开 Live |

---

## 1. 现场准备

**节点：** 引导节点。此时尚无 kubeconfig。

### 1.1 填写对照文件

先填完 `config.local.yaml`，再改 YAML。物理机现场核对盘符 WWN 时使用 `hosts` 段；这些 WWN 不写入共享 YAML。

| 变量 | 用途 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | 引导节点 TLS、引导集群 provider、Global Registration rootfs、Global 首次 Cluster/BaremetalCluster、Global KCP etcd SAN、installer `hostIP` |
| `<KIND_REGISTRY_ADDRESS>` | 引导集群 AppRelease `repoURL` / registry；物理机场景 Global SeedImage `baseImage` |
| `<PLATFORM_HOST>` | 成品 Global provider、installer console |
| `<PLATFORM_REGISTRY>` | installer registry、成品 Global provider、Workload 全部 registry。installer Success 之后才向这里推 |
| `<GLOBAL_CONTROL_PLANE_VIP>` | 引导集群 Bare Metal provider、Global BaremetalCluster、Global KCP etcd SAN、installer `features.ha.vip` |
| `<WORKLOAD_CLUSTER_NAME>` | Workload Cluster / BaremetalCluster / KCP / pool / Registration / SeedImage / MachineDeployment 的名字前缀。对照字段 `topology.workload_cluster_name`。本场示例 `workloadcluster`，下一场可另选，但不能是 `global`，且必须整包替换 |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | Workload BaremetalCluster、Workload KCP etcd SAN |
| `<NETWORK_DEVICE>` | 两个 BaremetalCluster 的 `spec.networkDevice`。Pool 条目不要写该字段 |
| `<GLOBAL_POD_CIDR>` `<GLOBAL_SERVICE_CIDR>` `<GLOBAL_KUBE_OVN_JOIN_CIDR>` | Global Cluster、installer JSON |
| `<WORKLOAD_POD_CIDR>` `<WORKLOAD_SERVICE_CIDR>` `<WORKLOAD_KUBE_OVN_JOIN_CIDR>` | Workload Cluster |
| `<SSH_AUTHORIZED_KEY>` | Global KCP、Workload KCP、ARM Worker KubeadmConfigTemplate；HCS cloud-init 的 `boot` 用户使用同一把 |
| `<KUBEADM_PROVIDER_CHART_VERSION>` | 引导集群 / 成品 Global 的 Kubeadm AppRelease。从 `.ALL.` 包文件名抄 |
| `<BAREMETAL_PROVIDER_MULTIARCH_CHART_VERSION>` | 引导集群 / 成品 Global 的 Bare Metal AppRelease。从 `.ALL.` 包文件名抄 |
| `<ACP_4_4_1_HYBRID_CORE_PACKAGE>` | Core Package 文件名 |
| `<KUBEADM_PROVIDER_ALL_PACKAGE>` | kubeadm provider `.ALL.` tgz 文件名 |
| `<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>` | Bare Metal provider `.ALL.` tgz 文件名 |
| `<K25_AMD64_IMAGE_TAG>` | 第 3 章导入 amd64 archive 后才能填 |
| `<K25_ARM64_IMAGE_TAG>` | installer Success 后导入 arm64 archive 后才能填 |
| `<GLOBAL_CP_INVENTORY_1/2/3>` | Global pool、导入清单。装机后再填实际名称 |
| `<WORKLOAD_CP_INVENTORY_1/2/3>` | Workload 控制面 pool。装机后再填 |
| `<WORKLOAD_ARM64_WORKER_INVENTORY>` | ARM Worker pool。装机后再填 |
| `registry` / `<BASE64_REGISTRY_PASSWORD>` | installer JSON 仓库账号。本场为 `registry` |
| `admin` / `<BASE64_PLATFORM_ADMIN_PASSWORD>` | installer JSON 平台账号，控制台登录同此 |

六段 CIDR 必须互不重叠，且不与管理网重叠。Join 使用独立网段，不要写成 Pod/Service 的子集。不要使用 `10.4.0.0/14`（会覆盖 `10.5` / `10.6`）。

### 1.2 交付物

从 Customer Portal 取 **同一套 ACP 4.4.1** 产物，放到引导节点（常见 `/root`）。压缩包不进 git。

两场景都需要：

| 包 | 对照字段 | 用途 |
|---|---|---|
| Core Package | `<ACP_4_4_1_HYBRID_CORE_PACKAGE>` | `setup.sh` 起 KIND `minialauda` + 嵌入式仓库 + `cpaas-installer` |
| kubeadm provider `.ALL.` tgz | `<KUBEADM_PROVIDER_ALL_PACKAGE>` | packtool 推进引导仓库，installer Success 后再推进平台仓库 |
| Bare Metal provider `.ALL.` tgz | `<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>` | 同上 |
| K25 amd64 OCI archive | `<ACP_4_4_1_K25_AMD64_ARCHIVE>` | `kubeos-rootfs` / `kubeos-iso`，本地 source tag `baremetal` |
| K25 arm64 OCI archive | `<ACP_4_4_1_K25_ARM64_ARCHIVE>` | installer Success 后才导入。不要在引导阶段覆盖 amd64 |

HCS 另备，上传为 HCS 系统模板：

| 包 | 文件名 | 平台模板名 |
|---|---|---|
| amd64 qcow2 | `kubeos:alaudaos-44.ku.202610100215-0.x86_64.qcow2` | `kubeos-202610100215-x86` |
| arm64 qcow2 | `kubeos:alaudaos-44.ku.202610100215-0.aarch64.qcow2` | `kubeos-202610100215-arm` |

启动方式必须为 **UEFI**。

### 1.3 对时

所有即将接入的物理机和 HCS 虚机，装系统前将时钟校准到彼此相差不超过 10 秒。裸金属在进入 Live ISO 前从 BMC/firmware 校准；HCS 虚机在开机前确认宿主机时间。漏做对时后再装机会卡住 Machine，处理见第 13 章。

### 1.4 SSH 公钥与 installer 密码

```bash
test -f ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

将这一整行填入 `<SSH_AUTHORIZED_KEY>`。私钥不要放入仓库。

installer JSON 的两个 password 字段必须是 **base64 编码后的字符串**，不是明文：

```bash
printf 'platform admin password: ' && read -rs PLAIN && echo
printf '%s' "$PLAIN" | base64 | tr -d '\n'; echo
# 填入 <BASE64_PLATFORM_ADMIN_PASSWORD>

printf 'registry password: ' && read -rs REGPLAIN && echo
printf '%s' "$REGPLAIN" | base64 | tr -d '\n'; echo
# 填入 <BASE64_REGISTRY_PASSWORD>
# 明文另行保存：第 3 章和推平台仓库时 ctr push 需要；不要提交
```

本场仓库用户名为 `registry`，平台用户名为 `admin`。不要与 Dex `admin@cpaas.io` 混用。

---

## 2. 用 Core Package 拉起 minialauda

**节点：** 引导节点。

将 Core Package 解到引导节点。本手册按 `/root/installer/setup.sh` 书写；现场目录不同则改路径。

```bash
test -x /root/installer/setup.sh
ls /root/installer/res/amd64/kind /root/installer/res/amd64/packtool /root/installer/res/amd64/nerdctl
cd /root/installer
bash setup.sh
```

`setup.sh` 会询问引导节点 IP、仓库是否设置密码。填入的 IP **必须是 `<BOOTSTRAP_HOST_IP>`**，整场不要更改。脚本完成后才有 kubeconfig。

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

成功条件：KIND 名为 `minialauda`；本机仓库 `127.0.0.1:11443`；`cpaas-installer` 在运行。默认仓库匿名时，不要创建空的 `global-registry-auth`。

裸金属场景的 SeedImage 需要 20Gi PVC。KIND 起来后立即确认存在默认 StorageClass。HCS 场景不创建 SeedImage，可跳过；若后续改走物理机 ISO 则会用到。

```bash
kubectl get sc
kubectl get sc -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{"\n"}{end}'
```

裸金属场景必须打出至少一个 StorageClass 名称。空则 ISO 永远没有 `downloadURL`。

`nerdctl` 不在 PATH 时：

```bash
export PATH="/root/installer/res/amd64:${PATH}"
command -v nerdctl || command -v docker
nerdctl ps --format '{{.Names}}' | grep minialauda || docker ps --format '{{.Names}}' | grep minialauda
```

---

## 3. 将 chart 与 amd64 OS 推入引导仓库

**节点：** 引导节点。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

本步只写入本机 `127.0.0.1:11443`。不要写入 `<KIND_REGISTRY_ADDRESS>` 或 `<PLATFORM_REGISTRY>`。此时平台仓库尚不存在。

### 3.1 packtool 推两个 provider chart

```bash
PACKTOOL=/root/installer/res/amd64/packtool
test -x "$PACKTOOL"
$PACKTOOL pack push -h
$PACKTOOL pack push -r "127.0.0.1:11443" -c /root/<KUBEADM_PROVIDER_ALL_PACKAGE>
$PACKTOOL pack push -r "127.0.0.1:11443" -c /root/<BAREMETAL_PROVIDER_ALL_MULTIARCH_PACKAGE>
```

先查看 help，不要假设认证开关名为 `-u`。引导仓库默认匿名。`setup.sh` 若已为仓库设置账号，按 help 中的认证参数再推。

### 3.2 导入 amd64 KubeOS，读取最终 tag，推入引导仓库

`/var/lib/containerd` 至少要有归档同等大小的空闲空间。不要先 `ctr images rm`：交接后还要从同一 namespace 再推到平台仓库。

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

最终 tag 以本次 `images ls` 和现场 archive 说明为准，填入 `config.local.yaml` 的 `<K25_AMD64_IMAGE_TAG>`，再替换 YAML。不要把 `baremetal` 写进 YAML。HCS 场景也要推 `kubeos-iso`：archive 内两份一起导入，catalog / rootfs 都使用这份 tag。

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

KIND 内通过 `<KIND_REGISTRY_ADDRESS>:11443` 读到的即刚才推进 `127.0.0.1:11443` 的同一份。装 OS 时走 `<BOOTSTRAP_HOST_IP>:11443`。

arm64 archive **本步不要导入**，会覆盖本地 `*:baremetal` source tag。

---

## 4. 在引导集群上安装 provider

KIND 名必须是 `minialauda`。

### 4.1 `global/bootstrap/01-dex-tls-bootstrap.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | Certificate `spec.commonName`、`spec.ipAddresses` |

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/01-dex-tls-bootstrap.yaml"
kubectl -n cpaas-system get certificate dex-tls-bootstrap
kubectl -n cpaas-system get secret dex.tls
```

`Ready=True` 后再安装 Bare Metal provider。该 Secret 不要导入成品 Global。

### 4.2 `global/bootstrap/02-kubeadm-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<KUBEADM_PROVIDER_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<KIND_REGISTRY_ADDRESS>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<KIND_REGISTRY_ADDRESS>:11443` |
| `<BOOTSTRAP_HOST_IP>` | `values.global.host`、`values.global.platformUrl`（`https://<BOOTSTRAP_HOST_IP>`） |

`values.global.auth.default_admin` 保持 `admin`。

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

引导阶段 catalog **只填 amd64**。不要在引导阶段加入 arm64 catalog。`default_admin` 保持 `admin`。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/03-baremetal-provider-apprelease.yaml"
kubectl -n cpaas-system get apprelease
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

catalog 必须有：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25: <KIND_REGISTRY_ADDRESS>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
```

两个 AppRelease Ready、catalog 已有 amd64 key 后再进入对应场景。

---

# 场景一：HCS 虚拟机 Bare Metal

HCS 虚拟机不支持挂载 ISO 启动，使用研发提供的 qcow2 系统模板。不要 apply 任何 `01-registration-seedimage-*.yaml`。

Global 控制面 Ready 后，交接、installer、平台仓库执行第 6–7 章（两场景共用）。然后回到第 8–9 章创建 Workload 虚机。

---

## 5. Global：接入三台控制面虚机并创建 Cluster

**节点：** 引导节点。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

### 5.1 创建 Global Registration

文件：`global/control-plane/01-registration-amd64.yaml`。对象名 `global-registration`。不要 apply 同目录的 `01-registration-seedimage-amd64.yaml`。名称保持 `global-registration`，以便与 DCS import 和 ISO 孪生文件对齐。

| 变量 | 位置 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `spec.config.elemental.install.system-uri` 的仓库主机 |
| `<K25_AMD64_IMAGE_TAG>` | 同一行的 tag |
| `/dev/vda` | `install.device`。HCS 虚机固定该值，不要写成 `/dev/elemental-install-target` |

```bash
grep -nE '<[^>]+>' "$PACK_DIR/global/control-plane/01-registration-amd64.yaml"
kubectl apply -f "$PACK_DIR/global/control-plane/01-registration-amd64.yaml"
kubectl -n cpaas-system get machineregistration global-registration -o yaml
```

`status.registrationURL` 非空后再 curl：

```bash
URL=$(kubectl -n cpaas-system get machineregistration global-registration \
  -o jsonpath='{.status.registrationURL}')
test -n "$URL"
curl -fsSk "$URL" > /root/registration-config.yaml
chmod 600 /root/registration-config.yaml
```

用该结果生成 cloud-init。样例：`config/cloud-init-user-data.example.yaml`。

```bash
cp "$PACK_DIR/config/cloud-init-user-data.example.yaml" /root/cloud-init-user-data.yaml
```

改两处，不要提交填好的文件：

- `ssh_authorized_keys`：换成第 1.4 节整行公钥
- `<REGISTRATION_CONFIG_YAML>`：换成 `/root/registration-config.yaml` 全文，按样例缩进贴入 `content: |` 下

`registrationURL` 与 curl 得到的 yaml 均为凭证。三台 Global x86 虚机共用这一份。

### 5.2 三台 Global HCS 虚机

**节点：** HCS 控制台。模板 `kubeos-202610100215-x86`，启动 **UEFI**。每台从干净模板新建，不要克隆已注册的磁盘。

高级选项：用户数据注入，文本，将 `/root/cloud-init-user-data.yaml` **全文**粘贴。三台使用同一份 Global amd64 cloud-init。开机后 cloud-init 执行 `elemental-register --install --no-toolkit`。网络须能访问引导节点 `:12443` 和 `:11443`。

系统盘默认 `/dev/vda`。qcow2 约 1.8 GB，HCS 盘常见 100 GB，空间不会自动扩展为根分区。SSH 进入后若 `df -h /` 仍很小：

```bash
lsblk -o NAME,PATH,SIZE,TYPE,FSTYPE,MOUNTPOINTS
findmnt -n -o SOURCE,FSTYPE,SIZE,AVAIL /
```

确认 `/` 在 `/dev/vda4`、ext4、盘尾仍有空闲后再扩（不要动 vda1 EFI）：

```bash
sfdisk --dump /dev/vda > /root/vda.sfdisk.bak
echo ", +" | sfdisk --no-reread -N 4 /dev/vda
partx -u /dev/vda
resize2fs /dev/vda4
```

已经顶到盘尾则跳过。分区号或文件系统不是 vda4/ext4 时，先改命令再执行。扩盘可在等待 Inventory 之前完成。

在虚机上查看注册：

```bash
sudo less /var/log/preinst-register.log
sudo journalctl -fu elemental-system-agent
```

### 5.3 确认虚机注册成功

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

三个名称必须互不相同，且三个 inventory `Ready=True`。将实际名称填入后续 pool。对照文件中的 UUID 仅为上一场示例，下一场不要复用。

### 5.4 `global/control-plane/02-control-plane-pool.yaml`

5.4–5.9 两场景共用。裸金属在三台 Inventory `Ready=True` 后从 10.4 跳到这里，按同一组文件 apply。

| 变量 | 字段 |
|---|---|
| `<GLOBAL_CP_INVENTORY_1>` | `spec.machineInventories[0].name` |
| `<GLOBAL_CP_INVENTORY_2>` | `spec.machineInventories[1].name` |
| `<GLOBAL_CP_INVENTORY_3>` | `spec.machineInventories[2].name` |

hostname 保持 `global-cp-1` / `global-cp-2` / `global-cp-3`。Pool 条目不要写 `networkDevice`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/02-control-plane-pool.yaml"
```

### 5.5 `global/control-plane/03-baremetal-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `metadata.annotations.cpaas.io/registry-address`，首次创建写成 `<BOOTSTRAP_HOST_IP>:11443` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `spec.controlPlaneLoadBalancer.host` |
| `<NETWORK_DEVICE>` | `spec.networkDevice` |

首次 apply 不要改成平台仓库。`type: External`、`port: 6443` 不要改。installer Success 后若该对象仍指向引导仓库，只按 7.1 patch，不要重新 apply 整个文件。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/03-baremetal-cluster.yaml"
```

### 5.6 `global/control-plane/04-control-plane-machine-template.yaml`

无变量。确认 `machineInventoryPoolRef.name` 为 `global-control-plane-pool`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/04-control-plane-machine-template.yaml"
```

### 5.7 `global/control-plane/05-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<BOOTSTRAP_HOST_IP>` | `metadata.annotations.cpaas.io/registry-address`，首次创建写成 `<BOOTSTRAP_HOST_IP>:11443` |
| `<GLOBAL_KUBE_OVN_JOIN_CIDR>` | `metadata.annotations.cpaas.io/kube-ovn-join-cidr` |
| `<GLOBAL_POD_CIDR>` | `spec.clusterNetwork.pods.cidrBlocks[0]` |
| `<GLOBAL_SERVICE_CIDR>` | `spec.clusterNetwork.services.cidrBlocks[0]` |

Kubernetes 版本保持 `v1.35.6-acp.9`，Kube-OVN 保持 `v4.4.12`。installer 会改写该 Cluster 的 registry annotation，成功后不要重新 apply 此文件。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/05-cluster.yaml"
```

### 5.8 `global/control-plane/06-kubeadm-control-plane.yaml`

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

kubelet patch 含 `imagePullCredentialsVerificationPolicy: NeverVerify`，从该字段到 `protectKernelDefaults` 再到 `tlsCertFile`，不要出现 `staticPodPath`。保留 init/join 的 `volume-plugin-dir` 与 controllerManager 的 `flex-volume-plugin-dir`。

```bash
kubectl apply -f "$PACK_DIR/global/control-plane/06-kubeadm-control-plane.yaml"
kubectl -n cpaas-system get cluster global
kubectl -n cpaas-system get kubeadmcontrolplane global-kcp
kubectl -n cpaas-system get machine,baremetalmachine -o wide
```

### 5.9 抽出 global-kubeconfig

此时仍不得拆除 KIND，也尚未 POST installer。

```bash
kubectl -n cpaas-system get secret global-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > "$PACK_DIR/global-kubeconfig"
chmod 600 "$PACK_DIR/global-kubeconfig"
export GLOBAL_KUBECONFIG="$PACK_DIR/global-kubeconfig"
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" get nodes -o wide
```

期待：三台 Master Node Ready；KCP `Ready=True`；Cluster `Phase: Provisioned`。然后进入第 6 章。

---

## 6. Global：交接与 installer

两场景共用本章。HCS 在第 5.9 节之后执行；裸金属在第 10.9 节之后执行。

**节点：** 引导节点。kubeconfig：`$BOOTSTRAP_KUBECONFIG`。

### 6.1 `global/bootstrap/04-dcs-import-extra-resources.yaml`

从 **已经绑定 Inventory 的** BaremetalMachine 抄名称，不要使用尚未被 Machine 认领的列表：

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

三个名字必须与 Pool 一致且互不相同。`global-registration` 已写死，不要改。不要向该文件加入 SeedImage、token、plan、bootstrap TLS。

```bash
kubectl apply -f "$PACK_DIR/global/bootstrap/04-dcs-import-extra-resources.yaml"
kubectl -n cpaas-system get cm dcs-import-extra-resources -o yaml
```

缺少该 ConfigMap 时，交接目标列表为空，成品 Global 没有 BaremetalMachine / Inventory / Registration。对象名 `dcs-import-extra-resources` 是 installer 接口要求，不要改名。

### 6.2 `global/bootstrap/05-installer-config.example.json`

复制到本机，不要改仓库里的 example：

```bash
cp "$PACK_DIR/global/bootstrap/05-installer-config.example.json" \
  /root/installer-config.json
```

| 变量 | JSON 字段 |
|---|---|
| `admin` | `basic.username` |
| `<BASE64_PLATFORM_ADMIN_PASSWORD>` | `basic.password` |
| `<PLATFORM_REGISTRY>` | `registry.domain`，写成 `<PLATFORM_REGISTRY>:11443` |
| `registry` | `registry.username` |
| `<BASE64_REGISTRY_PASSWORD>` | `registry.password` |
| `<PLATFORM_HOST>` | `console.host[0]`、`console.globalHost`。不要带 `https://`，不要带端口 |
| `<GLOBAL_POD_CIDR>` | `cluster.clusterCIDR` |
| `<GLOBAL_SERVICE_CIDR>` | `cluster.serviceCIDR` |
| `<GLOBAL_CONTROL_PLANE_VIP>` | `cluster.features.ha.vip` |
| `<BOOTSTRAP_HOST_IP>` | `hostIP` |

`deployMode` 保持 `normal`。`features.ha` 是 installer 接收 API VIP 的固定 schema。不要设置 `registry.externalAddress`。

```bash
python3 -m json.tool /root/installer-config.json >/dev/null
```

`basic.password` / `registry.password` 应为字母数字和 `=`，不是刚输入的明文。

```bash
kubectl -n cpaas-system get pods -l service_name=cpaas-installer -o wide
export INSTALLER_IP=$(kubectl -n cpaas-system get pods \
  -l service_name=cpaas-installer -o jsonpath='{.items[0].status.podIP}')
test -n "$INSTALLER_IP"
```

一个终端跟随日志：

```bash
kubectl -n cpaas-system logs -f \
  -l service_name=cpaas-installer --all-containers=true --prefix=true
```

另一个终端提交。必须在引导节点访问 installer Pod IP 的 `8080`，不要打平台 VIP。该 Pod IP 仅在引导集群网络内可达：

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

installer Success 不等于可以拆除引导集群。

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

三项均通过后再做第 7 章。KIND 先保留，引导节点还要用于 `ctr push`。

---

## 7. 推入平台仓库、成品 provider、catalog

两场景共用本章。

**节点：** 引导节点。OS 推送仍在引导节点执行。此后查看成品集群改用 `$GLOBAL_KUBECONFIG`。

引导仓库中的 ISO / catalog / chart **不会**随 installer 迁移。不推到平台仓库，Workload 会 `MANIFEST_UNKNOWN`。HCS 场景不创建 SeedImage，`kubeos-iso` 仍须推进平台仓库，与 archive 内 rootfs 使用同一套 tag。

### 7.1 确认并 patch registry 注解

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system \
  get cluster,baremetalcluster global -o \
  custom-columns='KIND:.kind,REG:.metadata.annotations.cpaas\.io/registry-address'
```

期待两边都是 `<PLATFORM_REGISTRY>:11443`。Cluster 仍为 `<BOOTSTRAP_HOST_IP>:11443` 时，先确认 installer Success。BaremetalCluster 官方不改写，按下面 patch：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system annotate baremetalcluster global \
  cpaas.io/registry-address=<PLATFORM_REGISTRY>:11443 --overwrite
```

不要出现 `127.0.0.1` / `<KIND_REGISTRY_ADDRESS>` / 残留的 `<BOOTSTRAP_HOST_IP>:11443`。不要重新 apply `03` 或 `05`。

### 7.2 推 amd64 OS 到平台仓库

按现场策略配置平台仓库证书信任，不要关闭 TLS 校验。用户名是 `registry`，密码是 1.4 生成的明文（不是 base64）。

```bash
export OS_TAG=<K25_AMD64_IMAGE_TAG>
export REGISTRY=<PLATFORM_REGISTRY>:11443
ctr -n bare-metal-os images ls -q | grep "$OS_TAG"
for repo in kubeos-rootfs kubeos-iso; do
  ctr -n bare-metal-os images push \
    -u "registry:<REGISTRY_PASSWORD>" \
    "$REGISTRY/tkestack/${repo}:${OS_TAG}" \
    "${repo}:${OS_TAG}"
done
```

### 7.3 导入 arm64 archive，推到平台仓库

amd64 最终 tag 已经打好后再导入 arm64，避免覆盖 `*:baremetal` 时把尚未 tag 的 amd64 弄丢。

```bash
ctr -n bare-metal-os images import --all-platforms --no-unpack /root/<ACP_4_4_1_K25_ARM64_ARCHIVE>
ctr -n bare-metal-os images ls -q
```

将读到的最终 tag 填入 `<K25_ARM64_IMAGE_TAG>`。两个最终 tag 不能互相覆盖。

```bash
export OS_TAG=<K25_ARM64_IMAGE_TAG>
export REGISTRY=<PLATFORM_REGISTRY>:11443
for repo in kubeos-rootfs kubeos-iso; do
  ctr -n bare-metal-os images tag \
    "${repo}:baremetal" "${repo}:${OS_TAG}"
  ctr -n bare-metal-os images push \
    -u "registry:<REGISTRY_PASSWORD>" \
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

认证参数按 help。仓库账号是 `registry`。

### 7.5 `global/control-plane/07-finished-kubeadm-provider-apprelease.yaml`

| 变量 | 字段 |
|---|---|
| `<KUBEADM_PROVIDER_CHART_VERSION>` | `spec.source.charts[0].targetRevision` |
| `<PLATFORM_REGISTRY>` | `spec.source.repoURL`、`values.global.registry.address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<PLATFORM_HOST>` | `values.global.host`、`values.global.platformUrl`（`https://<PLATFORM_HOST>`） |

该文件必须 apply 到成品 Global，不要 apply 到 `minialauda`。`default_admin` 保持 `admin`。

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

AppRelease 只是输入。Workload 真正读取的是该 ConfigMap：

```bash
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

`.data` 至少要有：

```text
v1.35.6-acp.9_amd64_alaudaos.k.25=<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>
v1.35.6-acp.9_arm64_alaudaos.k.25=<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>
```

缺 key 则 merge patch，不要 replace 整个 ConfigMap：

```bash
kubectl -n cpaas-system patch configmap elemental-image-catalog \
  --type merge \
  -p "{\"data\":{\"v1.35.6-acp.9_amd64_alaudaos.k.25\":\"<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_AMD64_IMAGE_TAG>\",\"v1.35.6-acp.9_arm64_alaudaos.k.25\":\"<PLATFORM_REGISTRY>:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>\"}}"
kubectl -n cpaas-system get configmap elemental-image-catalog \
  -o jsonpath='{.data}'; echo
```

ConfigMap 对象不存在时，先查 `08` 是否 Ready。两个 key 都正确后进入 7.8。裸金属还须确认成品 Global 有默认 StorageClass（后续 Workload SeedImage PVC）。

### 7.8 拆除 KIND

拆之前再核对 Cluster **和** BaremetalCluster 注解都是 `<PLATFORM_REGISTRY>:11443`。拆完后 `127.0.0.1` / `<KIND_REGISTRY_ADDRESS>` / `$BOOTSTRAP_KUBECONFIG` 失效，只保留 `$GLOBAL_KUBECONFIG`。

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system \
  get cluster,baremetalcluster global -o \
  custom-columns='KIND:.kind,REG:.metadata.annotations.cpaas\.io/registry-address'
export PATH="/root/installer/res/amd64:${PATH}"
kind delete cluster --name minialauda
```

`kind` 不在 PATH 且容器还在时：

```bash
nerdctl rm -f minialauda-control-plane 2>/dev/null || docker rm -f minialauda-control-plane
```

HCS 进入第 8 章。裸金属进入第 11 章。

---

## 8. Workload：amd64 控制面虚机

本章全部使用 `$GLOBAL_KUBECONFIG`。KIND 已拆除。不要复用 Global 的 Registration URL 或 cloud-init。不要 apply SeedImage。`<WORKLOAD_CLUSTER_NAME>` 在 apply 前整包替换，规则见第 0 章。

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl get nodes
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

### 8.1 创建 Workload 控制面 Registration

文件：`workload/control-plane/01-registration-amd64.yaml`。对象名 `<WORKLOAD_CLUSTER_NAME>-registration`。不要 apply 同目录的 `01-registration-seedimage-amd64.yaml`。

| 变量 | 位置 |
|---|---|
| `<PLATFORM_REGISTRY>` | `spec.config.elemental.install.system-uri` 的仓库主机，写成 `<PLATFORM_REGISTRY>:11443` |
| `<K25_AMD64_IMAGE_TAG>` | 同一行的 tag |

`install.device` 仍为 `/dev/vda`。

```bash
grep -nE '<[^>]+>' "$PACK_DIR/workload/control-plane/01-registration-amd64.yaml"
kubectl apply -f "$PACK_DIR/workload/control-plane/01-registration-amd64.yaml"
kubectl -n cpaas-system get machineregistration <WORKLOAD_CLUSTER_NAME>-registration -o yaml
```

```bash
URL=$(kubectl -n cpaas-system get machineregistration <WORKLOAD_CLUSTER_NAME>-registration \
  -o jsonpath='{.status.registrationURL}')
test -n "$URL"
curl -fsSk "$URL" > /root/registration-config-workload-cp.yaml
chmod 600 /root/registration-config-workload-cp.yaml
```

再从 `config/cloud-init-user-data.example.yaml` 复制一份，换成这一份 curl 结果和同一把 `<SSH_AUTHORIZED_KEY>`。HCS 模板仍是 `kubeos-202610100215-x86`。虚机网络走 `<PLATFORM_HOST>:443` 和 `<PLATFORM_REGISTRY>:11443`，不是引导节点 `:12443`。

### 8.2 三台 Workload 控制面虚机

从干净 x86 模板新建，UEFI，用户数据注入粘贴 Workload amd64 这份 cloud-init。三台共用同一份。可选扩 `/dev/vda4`，命令与 5.2 相同。

### 8.3 确认虚机注册成功

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

三个名称必须互不相同，且三个 inventory `Ready=True`。

### 8.4 `workload/control-plane/02-control-plane-pool.yaml`

8.4–8.8 两场景共用。裸金属在三台 Inventory `Ready=True` 后从 11.4 跳到这里。

| 变量 | 字段 |
|---|---|
| `<WORKLOAD_CP_INVENTORY_1>` | `spec.machineInventories[0].name` |
| `<WORKLOAD_CP_INVENTORY_2>` | `spec.machineInventories[1].name` |
| `<WORKLOAD_CP_INVENTORY_3>` | `spec.machineInventories[2].name` |

hostname 保持 `<WORKLOAD_CLUSTER_NAME>-cp-1/2/3`。三个 name 不能重复。若 API 报 `spec.machineInventories[2].name: Duplicate value`，只改第三个 name，不要删 Cluster。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/02-control-plane-pool.yaml"
```

### 8.5 `workload/control-plane/03-baremetal-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `metadata.annotations.cpaas.io/registry-address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | `spec.controlPlaneLoadBalancer.host` |
| `<NETWORK_DEVICE>` | `spec.networkDevice` |

不要写成引导仓库或 `127.0.0.1`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/03-baremetal-cluster.yaml"
```

### 8.6 `workload/control-plane/04-control-plane-machine-template.yaml`

对象名与 `machineInventoryPoolRef.name` 都带 `<WORKLOAD_CLUSTER_NAME>`。确认 pool 名为 `<WORKLOAD_CLUSTER_NAME>-control-plane-pool`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/04-control-plane-machine-template.yaml"
```

### 8.7 `workload/control-plane/05-cluster.yaml`

| 变量 | 字段 |
|---|---|
| `<PLATFORM_REGISTRY>` | `metadata.annotations.cpaas.io/registry-address`，写成 `<PLATFORM_REGISTRY>:11443` |
| `<WORKLOAD_KUBE_OVN_JOIN_CIDR>` | `metadata.annotations.cpaas.io/kube-ovn-join-cidr` |
| `<WORKLOAD_POD_CIDR>` | `spec.clusterNetwork.pods.cidrBlocks[0]` |
| `<WORKLOAD_SERVICE_CIDR>` | `spec.clusterNetwork.services.cidrBlocks[0]` |

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/05-cluster.yaml"
```

### 8.8 `workload/control-plane/06-kubeadm-control-plane.yaml`

| 变量 | 字段 |
|---|---|
| `<SSH_AUTHORIZED_KEY>` | `spec.kubeadmConfigSpec.users[0].sshAuthorizedKeys[0]` |
| `<WORKLOAD_CONTROL_PLANE_VIP>` | `clusterConfiguration.etcd.local.serverCertSANs` |

固定值：`replicas: 3`、`maxSurge: 0`、`version: v1.35.6-acp.9`。kubelet patch 含 `NeverVerify`，不要写 `staticPodPath`。

```bash
kubectl apply -f "$PACK_DIR/workload/control-plane/06-kubeadm-control-plane.yaml"
kubectl get cluster <WORKLOAD_CLUSTER_NAME>
kubectl get kubeadmcontrolplane <WORKLOAD_CLUSTER_NAME>-control-plane
kubectl get machines -l cluster.x-k8s.io/cluster-name=<WORKLOAD_CLUSTER_NAME> -o wide
```

---

## 9. Workload：arm64 Worker 虚机

不要使用 amd64 的 qcow2、Registration URL 或 cloud-init。不要 apply SeedImage。

### 9.1 创建 ARM Worker Registration

文件：`workload/worker/01-registration-arm64.yaml`。对象名 `<WORKLOAD_CLUSTER_NAME>-arm64-worker-registration`。不要 apply 同目录的 `01-registration-seedimage-arm64.yaml`。

| 变量 | 位置 |
|---|---|
| `<PLATFORM_REGISTRY>` | `spec.config.elemental.install.system-uri` 的仓库主机，写成 `<PLATFORM_REGISTRY>:11443` |
| `<K25_ARM64_IMAGE_TAG>` | 同一行的 tag |

```bash
grep -nE '<[^>]+>' "$PACK_DIR/workload/worker/01-registration-arm64.yaml"
kubectl apply -f "$PACK_DIR/workload/worker/01-registration-arm64.yaml"
kubectl -n cpaas-system get machineregistration <WORKLOAD_CLUSTER_NAME>-arm64-worker-registration -o yaml
```

```bash
URL=$(kubectl -n cpaas-system get machineregistration \
  <WORKLOAD_CLUSTER_NAME>-arm64-worker-registration \
  -o jsonpath='{.status.registrationURL}')
test -n "$URL"
curl -fsSk "$URL" > /root/registration-config-workload-arm.yaml
chmod 600 /root/registration-config-workload-arm.yaml
```

再从 `config/cloud-init-user-data.example.yaml` 复制一份，换成这一份 curl 结果和同一把 `<SSH_AUTHORIZED_KEY>`。HCS 模板用 `kubeos-202610100215-arm`，启动仍是 UEFI。

### 9.2 一台 ARM 虚机

从干净 ARM 模板新建。系统盘仍默认 `/dev/vda`，扩盘命令与 5.2 相同。

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

将实际名称填入后续 pool。名称不要与已有 inventory 重复，且 `Ready=True`。

### 9.3 `workload/worker/02-worker-pool-arm64.yaml`

9.3–9.6 两场景共用。裸金属在 ARM Inventory `Ready=True` 后从 12.3 跳到这里。

| 变量 | 字段 |
|---|---|
| `<WORKLOAD_ARM64_WORKER_INVENTORY>` | `spec.machineInventories[0].name` |

hostname 保持 `<WORKLOAD_CLUSTER_NAME>-arm64-worker-1`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/02-worker-pool-arm64.yaml"
```

### 9.4 `workload/worker/03-worker-machine-template-arm64.yaml`

对象名与 `machineInventoryPoolRef.name` 都带 `<WORKLOAD_CLUSTER_NAME>`。确认 pool 名为 `<WORKLOAD_CLUSTER_NAME>-arm64-worker-pool`。

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
imagePullCredentialsVerificationPolicy: NeverVerify
```

kubelet patch 不要写 `staticPodPath`。

```bash
kubectl apply -f "$PACK_DIR/workload/worker/04-worker-kubeadm-config-template-arm64.yaml"
```

### 9.6 `workload/worker/05-worker-machine-deployment-arm64.yaml`

对象名、`clusterName`、selector 均带 `<WORKLOAD_CLUSTER_NAME>`。固定值：

```text
replicas        1
maxSurge        0
maxUnavailable  1
version         v1.35.6-acp.9
```

```bash
kubectl apply -f "$PACK_DIR/workload/worker/05-worker-machine-deployment-arm64.yaml"
kubectl get machines -l cluster.x-k8s.io/cluster-name=<WORKLOAD_CLUSTER_NAME> -o wide
```

本场景到此结束。运维见第 13 章。

---

# 场景二：裸金属 Bare Metal

本场景全部为同网段物理机。不要 apply 仅 Registration 的 `01-registration-amd64.yaml` / `01-registration-arm64.yaml`。不要使用 qcow2 / cloud-init。

第 1–4 章完成后从第 10 章继续。kubeconfig 仍是 `$BOOTSTRAP_KUBECONFIG`。KIND 必须有默认 StorageClass。Pool / Cluster / KCP 的 apply 写在 5.4–5.9、8.4–8.8、9.3–9.6；那些小节虽在场景一标题下，裸金属同样执行。

Global 控制面 Ready 后执行第 6–7 章（与 HCS 同一套 YAML 和命令），再进入第 11 章。

---

## 10. Global：接入三台控制面物理机并创建 Cluster

### 10.1 `global/control-plane/01-registration-seedimage-amd64.yaml`

同时创建 `global-registration` 和 SeedImage。不要与 `01-registration-amd64.yaml` 一起 apply。

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

PVC Pending = 没有默认 StorageClass，不要空等。闸门是 `status.downloadURL` 非空，不是某个固定的 condition 名。

在 **引导节点** 下载 ISO（URL 通常只在集群网络内可访问）：

```bash
curl -fL --retry 3 -o /root/global-registration.iso "$ISO_URL"
ls -lh /root/global-registration.iso
```

将 ISO 拷到能挂这三台 BMC 虚拟光驱的机器。该 ISO 只给三台 Global 控制面，不要给 Workload 使用。`cleanupAfterMinutes: 120` 过后构建产物会被清理。

Registration URL 只在引导节点终端查看，不要写入仓库、工单、聊天：

```bash
kubectl -n cpaas-system get machineregistration global-registration \
  -o jsonpath='{.status.registrationURL}{"\n"}'
```

### 10.2 三台 Global 物理机装系统

**节点：** 每台 Global 物理机 BMC / Live ISO。不用 kubectl。

进入 Live 前确认这三台已经对时（见 1.3）。BMC 挂上 10.1 的 ISO，从光驱启动。Live 控制台一般是 root、无密码（以现场 ISO 提示为准）。

无 DHCP 时先配置静态 IP，再做软链。地址按 **这台机器** 修改：

```bash
ip -br link
ip -br addr
nmcli con add type ethernet ifname <这台机器网卡名> con-name mgmt \
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

有 DHCP 且能 ping 通引导节点则跳过配网。

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

不要链到 ISO、loop 或数据盘。软链完成后不要再手工执行 `elemental-register`。查看进度：

```bash
journalctl -f | grep -Ei 'elemental|register|install|cloud-init' || true
```

装完：弹出虚拟光驱，BMC 改为硬盘优先，确认下次进入已安装 OS。`/` 不能仍是 Live。已离开 Live 的机器不要再回 Live 重建链接。

### 10.3 采集 Inventory 名称

回到引导节点：

```bash
kubectl -n cpaas-system get machineinventory -o wide
```

三个 inventory `Ready=True` 后，将实际名称填入 `config.local.yaml` 的：

```text
<GLOBAL_CP_INVENTORY_1>
<GLOBAL_CP_INVENTORY_2>
<GLOBAL_CP_INVENTORY_3>
```

三个名称必须互不相同。主机须已从系统盘启动。此时 **不要** apply `04`。

### 10.4–10.9 创建 Pool / Cluster / KCP

与第 5.4–5.9 节使用同一组文件和同一组变量：

```text
global/control-plane/02-control-plane-pool.yaml
global/control-plane/03-baremetal-cluster.yaml
global/control-plane/04-control-plane-machine-template.yaml
global/control-plane/05-cluster.yaml
global/control-plane/06-kubeadm-control-plane.yaml
```

按 **5.4–5.9** 的占位符表、固定值和 apply 命令执行（含抽出 `global-kubeconfig`）。installer Success 后若 BaremetalCluster 仍指向引导仓库，按 **7.1** patch，不要重新 apply 整个 `03`。

然后进入第 6 章，再执行第 7 章。第 7.8 节拆 KIND 前，成品 Global 还需要默认 StorageClass（后续 Workload SeedImage PVC）。

---

## 11. Workload：amd64 控制面物理机

本章全部使用 `$GLOBAL_KUBECONFIG`。KIND 已拆除。不要复用 Global 的 ISO。Global 上要有默认 StorageClass。`<WORKLOAD_CLUSTER_NAME>` 规则见第 0 章。

```bash
export KUBECONFIG="$GLOBAL_KUBECONFIG"
kubectl get nodes
kubectl get sc
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

### 11.1 `workload/control-plane/01-registration-seedimage-amd64.yaml`

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
  ISO_URL=$(kubectl -n cpaas-system get seedimage <WORKLOAD_CLUSTER_NAME>-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
```

该 ISO 只给三台 Workload amd64 控制面。不要复用 Global ISO。

### 11.2 三台 Workload 控制面物理机

**节点：** 每台 Workload amd64 控制面 BMC / Live ISO。

进入 Live 前对时。WWN 对照：

```text
<WORKLOAD_CP_1_OS_DISK_WWN>
<WORKLOAD_CP_2_OS_DISK_WWN>
<WORKLOAD_CP_3_OS_DISK_WWN>
```

symlink、无 DHCP 配网、弹出光驱、改硬盘优先，流程与 10.2 相同。网络须能访问 `<PLATFORM_HOST>` 的 443 和 `<PLATFORM_REGISTRY>:11443`，不是引导节点 `:12443`。

### 11.3 采集 Inventory 名称

```bash
kubectl -n cpaas-system get machineinventory \
  -o custom-columns='NAME:.metadata.name,READY:.status.ready'
```

将三个互不相同的实际名称填入：

```text
<WORKLOAD_CP_INVENTORY_1>
<WORKLOAD_CP_INVENTORY_2>
<WORKLOAD_CP_INVENTORY_3>
```

### 11.4–11.8 创建 Pool / Cluster / KCP

与第 8.4–8.8 节使用同一组文件和同一组变量：

```text
workload/control-plane/02-control-plane-pool.yaml
workload/control-plane/03-baremetal-cluster.yaml
workload/control-plane/04-control-plane-machine-template.yaml
workload/control-plane/05-cluster.yaml
workload/control-plane/06-kubeadm-control-plane.yaml
```

按 **8.4–8.8** 的占位符表、固定值和 apply 命令执行。三个 pool inventory name 不能重复；若 API 报 Duplicate value，只改第三个 name，不要删 Cluster。

---

## 12. Workload：arm64 Worker 物理机

不要使用 amd64 的 ISO、tag 或 Registration。

### 12.1 `workload/worker/01-registration-seedimage-arm64.yaml`

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
    <WORKLOAD_CLUSTER_NAME>-arm64-worker-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
```

### 12.2 ARM 物理机

**节点：** 那一台 ARM Worker BMC / Live ISO。

进入 Live 前对时。WWN 对照 `<WORKLOAD_ARM64_WORKER_1_OS_DISK_WWN>`。symlink 和配网流程与 10.2 相同。

装完后将实际 inventory 名称填入 `<WORKLOAD_ARM64_WORKER_INVENTORY>`。

### 12.3–12.6 创建 Worker Pool / Template / MD

与第 9.3–9.6 节使用同一组文件：

```text
workload/worker/02-worker-pool-arm64.yaml
workload/worker/03-worker-machine-template-arm64.yaml
workload/worker/04-worker-kubeadm-config-template-arm64.yaml
workload/worker/05-worker-machine-deployment-arm64.yaml
```

按 **9.3–9.6** 执行（含 `volume-plugin-dir`、`NeverVerify`、`replicas: 1` / `maxSurge: 0`）。

本场景到此结束。运维见第 13 章。

---

## 13. 运维

先看日志，再删除单个对象。删除 `Cluster` 是最后手段，且只能删 Workload，不能删 `Cluster/global`。

### 13.1 时间同步

漏做对时：先将相关物理机或虚机时间同步，再删除卡住的 `Machine`，观察控制器是否重建。

```bash
date -u
timedatectl show -p NTPSynchronized --value
chronyc tracking
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system get machine -o wide
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system delete machine <FAILED_MACHINE_NAME>
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" -n cpaas-system get machine,baremetalmachine -o wide \
  -l cluster.x-k8s.io/cluster-name=<CLUSTER_NAME>
```

### 13.2 在引导节点查看 installer

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

### 13.3 触发重新安装

资源已经 apply 完但卡住时，先删除失败的 `Machine`：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system get machine,baremetalmachine -o wide
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system delete machine <FAILED_MACHINE_NAME>
```

旧对象被清掉、新对象被拉起，说明上一轮已经清理并开始重装。

`ImageCatalogMiss`：先按第 7.7 节补 catalog key，再只删失败的 `BaremetalMachine`。不要删 Cluster。

只有单个对象删除后仍无法拉起、并且现场允许重建该 Workload 时，才删除 Workload Cluster：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system delete cluster <WORKLOAD_CLUSTER_NAME>
```

等待级联对象清理完成后，再按顺序 apply：

```text
workload/control-plane/03-baremetal-cluster.yaml
workload/control-plane/04-control-plane-machine-template.yaml
workload/control-plane/05-cluster.yaml
workload/control-plane/06-kubeadm-control-plane.yaml
workload/worker/03-worker-machine-template-arm64.yaml
workload/worker/04-worker-kubeadm-config-template-arm64.yaml
workload/worker/05-worker-machine-deployment-arm64.yaml
```

pool 也被清掉时，先确认 inventory Ready，再 apply 两个 `02`。已经存在的 Registration / SeedImage / inventory 不要重复创建。不要删除 `Cluster/global`。

HCS 场景重建虚机时，从干净 qcow2 模板再开一台，粘贴同一角色的 cloud-init，不要克隆已注册磁盘。

### 13.4 登录机器查看 cloud-init / elemental-agent

HCS 虚机额外看预安装日志；根分区未自动扩大时，先 `lsblk` / `findmnt` 确认 `/` 在 `/dev/vda4` 且为 ext4，再按 5.2 扩盘。不要扩 EFI 分区。

```bash
sudo less /var/log/cloud-init-output.log
sudo less /var/log/preinst-register.log
sudo journalctl -fu elemental-system-agent
```

unit 名如果是 `elemental-agent-system`：

```bash
sudo journalctl -fu elemental-agent-system
```

不确定时先查：

```bash
systemctl list-units --type=service --all | grep -Ei 'elemental|cloud-init'
```

Live ISO 还不能 SSH 时，在 BMC 上查看：

```bash
journalctl -f | grep -Ei 'elemental|register|install|cloud-init' || true
```

### 13.5 kubelet / containerd

```bash
sudo journalctl -fu kubelet
sudo journalctl -fu containerd
sudo crictl ps -a
sudo systemctl status kubelet containerd
```

节点上如果已经出现 `"staticPodPath": null`，先改对应 YAML，再按 13.3 重建那台 Machine。

### 13.6 查看 cluster-transformer

installer 尚未 Success 时，看引导集群：

```bash
kubectl --kubeconfig "$BOOTSTRAP_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cluster-transformer \
  --all-containers=true --prefix=true
```

installer Success 之后，看成品 Global：

```bash
kubectl --kubeconfig "$GLOBAL_KUBECONFIG" \
  -n cpaas-system logs -f -l service_name=cluster-transformer \
  --all-containers=true --prefix=true
```

installer 负责 Global 首次安装；transformer 负责后续插件和集群变更。
