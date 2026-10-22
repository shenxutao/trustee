# Trustee Helm Chart（Trustee Helm 图表）

> 说明：本文件是 `deployment/helm-chart/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

> 本文件是 [README.md](./README.md) 的简体中文译文，仅用于阅读参考；配置键、默认值、命令、链接与代码块均与原文保持一致。请勿据本文件反向生成英文原文。

用于在 Kubernetes 上部署 [Confidential Containers](https://github.com/confidential-containers) **Trustee** 的 Helm chart：**KBS**、**gRPC AS** 和 **RVPS**，并可选捆绑部署 **PostgreSQL**（[Bitnami chart](https://artifacthub.io/packages/helm/bitnami/postgresql)）与 **Valkey**（[Bitnami chart](https://artifacthub.io/packages/helm/bitnami/valkey)，用于 KBS 会话的 Redis 协议存储）。KBS 连接到远程的 **`coco_as_grpc`** Attestation Service（AS）。

## 安装

**要求**：Kubernetes 1.19+、Helm 3。如果需要捆绑部署的 Postgres（即当 **`storageBackend.type: Postgres`** 或 **`sessionStorageType: Postgres`** 时），Bitnami 子 chart 使用由 PVC 支撑的存储，因此你的集群必须提供可用的 **StorageClass**（或者你必须绑定一个已有的卷声明）。

从**仓库根目录**执行：

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace
```

等待工作负载就绪，然后端口转发 KBS（默认 HTTP **8080**）。内部的 ClusterIP Service 名称为 **`<Helm fullname>-kbs`**（按下面的安装命令即为 **`trustee-kbs`**）：

```bash
kubectl get pods -n coco-trustee -w
kubectl port-forward -n coco-trustee svc/trustee-kbs 8080:8080
```

卸载该 release：

```bash
helm uninstall trustee -n coco-trustee
```

> [!NOTE]
> 当 `secrets.useEphemeralGeneratedKeys` 为 `true`（默认值）时，一个 **post-delete** Helm hook 会自动删除该 release 作用域下的 `*-bootstrap-user-keys` Secret。

## 典型场景

### 默认：LocalFs 存储

与上文**安装**一节相同。如果 **`storageBackend.type`** 与 **`sessionStorageType`** 都不是 **`Postgres`**，该 chart 不会部署捆绑的 Postgres；各组件使用默认的 **`storageBackend`**（例如 **LocalFs**）。

### 以 PostgreSQL 作为存储后端 + KBS 会话存于内存

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/postgres-backend.yaml
```

这将启用 **Bitnami PostgreSQL** 子 chart（`postgresql.enabled: true`）并设置 **`storageBackend.type: Postgres`**。KBS 会话仍保留在内存中（`sessionStorageType: Memory`）。演示用凭据默认为 `trustee` / `trustee` / `trustee`（可通过 `postgresql.auth.*` 覆盖）。

### 外部 PostgreSQL

当使用外部 Postgres 服务时，设置 **`storageBackend.postgres.mode=external`**，预先创建一个带有 **`POSTGRES_URL`** 键的 Secret，并让 chart 指向它：

```bash
kubectl create secret generic trustee-external-postgres -n coco-trustee \
  --from-literal=POSTGRES_URL='postgresql://user:password@postgres.example.com:5432/trustee?sslmode=require'

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  --set storageBackend.type=Postgres \
  --set storageBackend.postgres.mode=external \
  --set storageBackend.postgres.external.existingSecretName=trustee-external-postgres \
  --set storageBackend.postgres.external.existingSecretKey=POSTGRES_URL
```

当 `storageBackend.postgres.mode=external` 时，chart **不会**部署 Bitnami 子 chart（`postgresql.enabled` 保持 `false`），即使 `storageBackend.type` 或 `sessionStorageType` 要求使用 Postgres。

### 使用 Valkey（Redis 协议）存储 KBS 会话

KBS 的 **`Redis`** 会话后端使用 Redis 有线协议。该 chart 捆绑的是 **Valkey**（BSD 许可）而非 Redis——后者的许可证已不再被 OSI 认可；任何兼容 Redis 协议的服务都可以使用。将会话存放在 KBS Pod 之外，可以运行多个 KBS 副本。

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/valkey-sessions.yaml
```

这将启用 **Bitnami Valkey** 子 chart（`valkey.enabled: true`）并设置 **`sessionStorageType: Redis`**。chart 会把连接 URL 写入一个 release 作用域的 Secret，并以 **`REDIS_URL`** 的形式注入 KBS。演示用密码默认为 `trustee`（可通过 `valkey.auth.password` 覆盖）。会话是短期的，因此捆绑的 Valkey 默认以 `standalone` 模式运行且不带 PVC（`valkey.primary.persistence.enabled: false`）。

默认的 Valkey 镜像从 **docker.io** 拉取，而该仓库对匿名拉取有速率限制。可以覆盖镜像来源以使用私有镜像仓库：

```bash
helm upgrade --install trustee ./deployment/helm-chart ... \
  -f ./deployment/helm-chart/scenarios/valkey-sessions.yaml \
  --set valkey.image.registry=mirror.example.com \
  --set valkey.image.repository=bitnami/valkey \
  --set valkey.image.tag=9.1.0
```

（chart 级别的 `global.imageRegistry` 也同样会被 Bitnami 子 chart 采纳。）

### 外部兼容 Redis 的服务

当使用外部兼容 Redis 的服务时，设置 **`storageBackend.redis.mode=external`**，预先创建一个存放连接 URL 的 Secret，并让 chart 指向它：

```bash
kubectl create secret generic trustee-external-redis -n coco-trustee \
  --from-literal=REDIS_URL='redis://:password@redis.example.com:6379'

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  --set sessionStorageType=Redis \
  --set storageBackend.redis.mode=external \
  --set storageBackend.redis.external.existingSecretName=trustee-external-redis \
  --set storageBackend.redis.external.existingSecretKey=REDIS_URL
```

当 `storageBackend.redis.mode=external` 时，chart **不会**部署 Valkey 子 chart（`valkey.enabled` 保持 `false`）。

### 自带密钥（BYOK）

密钥材料仅由 **`secrets.useEphemeralGeneratedKeys`** 控制：

- **`true`（默认）：** 一个 Helm **pre-install / pre-upgrade hook** Job 会生成一次性的演示密钥，并写入一个 release 作用域的 Secret（名称以 **`bootstrap-user-keys`** 结尾）。**`helm uninstall`** 会触发一个 **post-delete** hook 来删除该 Secret。
- **`false`：** 你必须在目标命名空间中**预先创建**一个 Kubernetes **`Secret`**，然后把 **`secrets.existingSecretName`** 设为该名称。此时 bootstrap hook **不会**被渲染。

启用一次性生成时，该 hook 使用：

- 一个 `initContainer`（OpenSSL 镜像），用于把密钥生成到 `emptyDir` 中；
- 一个 `quay.io/kata-containers/kubectl` 容器，用于根据生成的文件创建 Secret。

这两个镜像都可以通过 `bootstrapUserKeysJob.keygenImage.*` 与 `bootstrapUserKeysJob.kubectlImage.*` 覆盖。

停用一次性生成时，该 Secret 必须定义下列**数据键**（值为 PEM 文本或 base64 编码的 PEM，与任何 `kubectl create secret generic --from-file=...` 相同）：

| Secret 键 | 作用 |
|------------|------|
| **`KBS_ADMIN_PRIVATE_KEY`** / **`KBS_ADMIN_PUBKEY`** | KBS admin API 的 Ed25519 密钥对（用于签发 admin JWT）。 |
| **`KBS_ADMIN_TOKEN`** | 供 `kbs-client --admin-token-file` 使用的预签发 admin bearer JWT（启用一次性密钥时由 bootstrap hook 生成）。 |
| **`AS_TOKEN_SIGNING_PRIVATE_KEY`** | Attestation Service：用于签发 attestation token。 |
| **`AS_TOKEN_VERIFICATION_PUBLIC_KEY_CERT_CHAIN`** | AS：`x5c` / 证书链；KBS：用于校验 token 的信任锚。 |

chart 会把该 Secret 挂载到 KBS 与 gRPC AS 上，并将这些键**映射**为容器内 **`/opt/confidential-containers/kbs/user-keys`** 下的路径 **`private.key`**、**`public.pub`**、**`token.key`**、**`token-cert-chain.pem`**。

示例（先创建 Secret，再安装）：

```bash
kubectl create secret generic trustee-byok-keys -n coco-trustee \
  --from-file=KBS_ADMIN_PRIVATE_KEY=./admin.key.pem \
  --from-file=KBS_ADMIN_PUBKEY=./admin.pub.pem \
  --from-file=AS_TOKEN_SIGNING_PRIVATE_KEY=./token.key.pem \
  --from-file=AS_TOKEN_VERIFICATION_PUBLIC_KEY_CERT_CHAIN=./token-chain.pem

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  --set secrets.useEphemeralGeneratedKeys=false \
  --set secrets.existingSecretName=trustee-byok-keys
```

或者直接使用 **`scenarios/bring-your-own-keys.yaml`**（在该文件的注释中调整 Secret 名称 / 文件路径）。

### KBS 原生 HTTPS

默认的 KBS 监听器使用明文 HTTP。如果客户端或机密 guest 会直接访问该 KBS 端点，请启用原生 HTTPS，并提供一个已有的、包含端点私钥与证书链的 Secret。chart 不会生成这些身份材料，因为其证书的 SAN 必须匹配 KBS 客户端所使用的地址。

```bash
kubectl create namespace coco-trustee
kubectl create secret tls trustee-kbs-tls \
  --namespace coco-trustee \
  --key ./kbs.key \
  --cert ./kbs.crt

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee \
  -f ./deployment/helm-chart/scenarios/native-tls.yaml
```

当 `kbs.tls.enabled=true` 时，chart 会让 `insecure_http` 保持其安全的默认值（`false`），从所选的 Kubernetes TLS Secret 中挂载标准的 `tls.key` 与 `tls.crt` 数据键，并把 KBS 的健康探针改为 HTTPS。

KBS 不会动态重新加载端点身份材料，因此在替换 TLS Secret 内容后，请重启 KBS Deployment。

Ingress TLS 终结是另一种模式：若应由 Ingress controller 提供 HTTPS 并把明文 HTTP 转发给 KBS，请保持 KBS 原生 TLS 关闭，并配置 `ingress.tls`。除非所选的 Ingress controller 已明确配置为对其后端连接使用 HTTPS，否则不要将 KBS 原生 TLS 与 Ingress 同时使用。

### 按客户端限流

KBS 期望在其前置代理上做按客户端的限流；原因参见 [Rate Limiting](../../kbs/docs/rate-limiting.md)。使用本 chart 时，请在 Ingress controller 上配置该限额，并通过 `ingress.annotations` 传递它所需的注解。Ingress API 与 Gateway API 都无法表达限流，因此请遵循该 controller 自身的文档。chart 通过一条 Ingress 规则路由整个 KBS API，所以该限额覆盖所有 `/kbs/v0` 路径，KBS 侧无需任何配置。

### IBM Secure Execution（s390x，IBM 安全执行）

在 **s390x** 上，**IBM Secure Execution（SE）** verifier 在运行时需要 attestation 材料。由于 KBS 与**远程的 `coco_as_grpc` AS** 通信，该 verifier 运行在 **AS Pod** 内，因此这些材料必须挂载到 **AS** 而不是 KBS。（这与 `kbs/config/kubernetes/overlays/ibm-se` 中内置 AS 的 kustomize overlay 不同，后者把它们挂载在 KBS 上。）

verifier 从 **`/run/confidential-containers/ibmse/`** 下的固定路径读取材料（可通过 `SE_*` 环境变量覆盖；参见 `deps/verifier/src/se/README.md`）。chart 通过 PersistentVolume / PersistentVolumeClaim 从**本地节点路径**挂载它们——把 **`as.verifier.se.credsDir`** 设为节点上存放这些材料的目录（相当于 kustomize overlay 中使用的 `IBM_SE_CREDS_DIR`），并把 **`as.verifier.se.nodeName`** 设为该节点的名称。随后 chart 会创建一个 `local` 类型的 PV + PVC，并在 AS Pod 上把整个目录挂载到 `/run/confidential-containers/ibmse/`。

| 材料 | 在 `credsDir` 下的预期路径 | 说明 |
|----------|-------------------------------|-------|
| RSA measurement 密钥对 | `rsa/encrypt_key.{pem,pub}` | 私钥**敏感**——请限制节点访问。 |
| 签名 / 中间证书 | `certs/` | **目录**；其中所有文件都会被读取。 |
| CRL | `crls/` | **目录**；其中所有文件都会被读取。 |
| Host Key Documents（HKD） | `hkds/` | **目录**；其中所有文件都会被读取。 |
| SE 镜像头 | `hdr/hdr.bin` | 二进制文件。 |
| 根 CA（可选） | `root_ca.crt` | 单个文件。 |

设置 **`CERTS_OFFLINE_VERIFICATION=true`**（通过 `as.extraEnvVars`）以离线校验 HKD 证书链。除开发环境外，**不要**设置 `SE_SKIP_CERTS_VERIFICATION=true`——它会关闭 HKD 证书链校验。

```bash
# 1. 把所有材料放到目标 s390x 节点上的某个目录中，例如：
#    $IBM_SE_CREDS_DIR/{rsa/,certs/,crls/,hkds/,hdr/hdr.bin}
#    如何获取这些材料参见 deps/verifier/src/se/README.md。

# 2. 安装，让 chart 指向该节点与目录：
helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/ibm-se.yaml \
  --set as.verifier.se.credsDir=$IBM_SE_CREDS_DIR \
  --set as.verifier.se.nodeName=<your-s390x-node-name>
```

请使用以 `se-verifier` feature 构建的 **s390x** AS 镜像（`as.image.repository` / `as.image.tag`）。完整的覆盖项参见 **`scenarios/ibm-se.yaml`**。之后按 `deps/verifier/src/se/README.md` 中的说明设置 SE attestation 策略。

### AMD SEV-SNP 离线 VCEK 存储

**SNP** verifier 需要一个 **VCEK** 证书来校验 attestation report。默认情况下它会从 **AMD KDS** 获取，这要求 AS Pod 具备出网连通性。气隙集群可以改为把证书预置在节点上，让 verifier 在本地读取。

把 **`as.verifier.snp.kdsStoreHostPath`** 设为节点上存放该存储的目录，把 **`as.verifier.snp.nodeName`** 设为该节点的名称，并向 **`as.verifier.snp.vcekSources`** 中添加一条 **`OfflineStore`** 条目。随后 chart 会创建一个 `local` 类型的 PV + PVC，并在 AS Pod 上把该目录挂载到 `/opt/confidential-containers/attestation-service/kds-store`。这三个值必须同时提供：如果挂载了存储却没有 `OfflineStore` 来源，或者配置了 `OfflineStore` 来源却没有挂载，渲染都会失败，因此误配置不会静默回退到 KDS。

```bash
# 1. 把证书放到目标 SNP 节点上的某个目录中，布局如下
#    $KDS_STORE/vcek/{hwid}/{tcb_prefix}_vcek.der  (preferred)
#    $KDS_STORE/vcek/{hwid}/vcek.der               (fallback)

# 2. 安装，让 chart 指向该节点与目录：
helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  --set as.verifier.snp.kdsStoreHostPath=$KDS_STORE \
  --set as.verifier.snp.nodeName=<your-snp-node-name> \
  --set 'as.verifier.snp.vcekSources[0].type=OfflineStore'
```

各来源按给定顺序依次尝试，因此追加 `--set 'as.verifier.snp.vcekSources[1].type=KDS'` 可保留 AMD KDS 作为本地缺失证书的回退。关于如何构建 `vcek/` 目录以及何时需要刷新，参见 [the AMD offline certificate cache guide](../../attestation-service/docs/amd-offline-certificate-cache.md)。

### SPIRE 插件

**`spire`** KBS 插件通过既有 SPIRE 部署的 Delegated Identity API，向已通过 attestation 的 guest 分发 SPIFFE X.509-SVID。它要求 KBS 镜像以 `spire-plugin` cargo feature 构建，并且集群中已经运行着 SPIRE server + agent，且启用了 admin（Delegated Identity API）socket 并挂载到宿主机上——**本 chart 自身不部署 SPIRE**。

Delegated Identity API 提供在与常规 Workload API socket 不同的独立 admin socket 上，因此它不会通过 `csi.spiffe.io` CSI driver 暴露。它需要启用 SPIRE agent 的 admin socket 并挂载到宿主机（上游 SPIRE Helm chart 中的 `spire-agent.sockets.admin.enabled` / `mountOnHost`），随后 KBS 通过 `kbs.extraVolumes` / `kbs.extraVolumeMounts` 以 `hostPath` 方式挂载它。这要求 KBS 与 SPIRE agent 运行在同一节点上。

```bash
helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/spire.yaml
```

完整的覆盖项参见 **`scenarios/spire.yaml`**，插件细节（端点、selector 映射、配置）参见 `kbs/docs/plugins/spire.md`。

## 测试

**查看资源**：

```bash
kubectl get deploy,pods,svc -n coco-trustee
helm status trustee -n coco-trustee
```

**仅渲染检查**（不安装）：

```bash
helm dependency update ./deployment/helm-chart

helm template trustee ./deployment/helm-chart \
  -f ./deployment/helm-chart/scenarios/postgres-backend.yaml \
  --namespace coco-trustee > /tmp/trustee-render.yaml
```

如果你的集群无法从 Pod 解析 `*.svc.cluster.local`，请在覆盖 values 中设置 `dnsHostAliasWorkaround: true`，然后在 Service 已经存在之后再运行一次 `helm upgrade`，以便 Helm 的 `lookup` 能够解析 ClusterIP。

**kbs-client**（从仓库构建：`cargo build -p kbs-client --release`）：在使用一次性密钥时，由 hook 创建的 Secret（名称以 **`bootstrap-user-keys`** 结尾）在 **`KBS_ADMIN_TOKEN`** 下包含一个预签发的 admin JWT。KBS 期望 `authorization_mode = "AuthenticatedAuthorization"`，并带有一个包含 **`role`** 声明、且与 `[admin.authorization.regex_acl]` 匹配的 bearer JWT（默认角色为 **`admin`**）。

```bash
kubectl port-forward -n coco-trustee svc/trustee-kbs 8080:8080 &
SECRET=$(kubectl get secrets -n coco-trustee -o name | grep bootstrap-user-keys | head -1 | cut -d/ -f2)
kubectl get secret "$SECRET" -n coco-trustee -o jsonpath='{.data.KBS_ADMIN_TOKEN}' | base64 -d >/tmp/admin-token
kbs-client --url http://127.0.0.1:8080 config --admin-token-file /tmp/admin-token set-resource-policy --allow-all
```

设置一个机密资源（先 `config` + `--admin-token-file`，再 `set-resource`）：

```bash
echo 'demo-payload' >/tmp/demo-resource.txt
kbs-client --url http://127.0.0.1:8080 config --admin-token-file /tmp/admin-token set-resource \
  --path my_repo/resource_type/demo --resource-file /tmp/demo-resource.txt
```

按 KBS URI 路径获取资源（`get-resource` 是顶层子命令；它遵循你所用客户端构建与策略对应的常规 attestation / token 流程）：

```bash
kbs-client --url http://127.0.0.1:8080 get-resource --path my_repo/resource_type/demo
```

## 配置

默认的 **`values.yaml`** 有意保持精简。**LocalFs** / **LocalJson** 的固定落盘路径定义在 **`templates/_helpers.tpl`** 中（不可通过 values 覆盖）。你仍然可以用 `-f` / `--set` 合并额外的键（Helm 会合并任意 values）。

## 配置值

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| as.affinity | object | `{}` | AS Pod 的亲和性与反亲和性调度规则。 |
| as.extraEnvVars | list | `[]` | AS 容器的额外环境变量（例如 `HTTP(S)_PROXY` 与 `NO_PROXY`）。 |
| as.image.pullPolicy | string | `"Always"` | AS 容器镜像拉取策略。 |
| as.image.repository | string | `"ghcr.io/confidential-containers/staged-images/coco-as-grpc"` | AS 容器镜像仓库。 |
| as.image.tag | string | `"latest"` | AS 容器镜像标签。 |
| as.imagePullSecrets | list | `[]` | 私有镜像仓库可选的镜像拉取 Secret。 |
| as.nodeSelector | object | `{}` | AS Pod 的节点标签选择约束。 |
| as.podAnnotations | object | `{}` | 额外的 Pod 注解。 |
| as.podSecurityContext | object | `{}` | Pod 级别 security context 覆盖项。 |
| as.replicaCount | int | `1` | Attestation Service Pod 副本数。 |
| as.resources | object | `{"limits":{"cpu":"4","memory":"4Gi"},"requests":{"cpu":"500m","memory":"1Gi"}}` | AS 的容器 CPU/内存 request 与 limit。 |
| as.service.loadBalancerAnnotations | object | `{}` | 当 `as.service.type` 为 `LoadBalancer` 时应用的注解。 |
| as.service.port | int | `50004` | AS Service 端口。 |
| as.service.type | string | `"ClusterIP"` | AS Service 类型（`ClusterIP` 或 `LoadBalancer`）。 |
| as.tolerations | list | `[]` | 用于将 AS Pod 调度到带污点节点上的容忍项。 |
| as.verifier.dcap.collateral_service | string | `"https://api.trustedservices.intel.com/sgx/certification/v4/"` | Intel DCAP collateral service URL。配置了 `as.verifier.dcap` 时为必填。 |
| as.verifier.dcap.tcb_update_type | string | `"early"` | DCAP TCB 更新类型（例如 `early`）。 |
| as.verifier.nvidia.type | string | `"Local"` | NVIDIA verifier 类型：`Local` 或 `Remote`。为 `Remote` 时必须设置 `verifierUrl`。 |
| as.verifier.nvidia.verifierUrl | string | `"https://nras.attestation.nvidia.com/v4/attest"` | 当 `as.verifier.nvidia.type` 为 `Remote` 时的 NRAS URL。 |
| as.verifier.se.credsDir | string | `""` | 目标节点上存放 IBM SE attestation 材料的目录（`rsa/`、`certs/`、`crls/`、`hkds/`、`hdr/hdr.bin`）的绝对路径。非空时，chart 会创建一个 `local` 类型的 PersistentVolume + PersistentVolumeClaim，并在 AS Pod 上把该目录挂载到 `/run/confidential-containers/ibmse/`。需要配合 `as.verifier.se.nodeName`。 |
| as.verifier.se.nodeName | string | `""` | 存放 IBM SE 材料目录（`as.verifier.se.credsDir`）所在的 Kubernetes 节点名称。设置了 `as.verifier.se.credsDir` 时为必填；用于 PersistentVolume 的 `nodeAffinity`。 |
| as.verifier.snp.kdsStoreHostPath | string | `""` | 目标节点上存放 AMD SNP 离线 VCEK 证书存储的目录（必须包含 `vcek/` 子目录）的绝对路径。非空时，chart 会创建一个 `local` 类型的 PV + PVC，并在 AS Pod 上把它挂载到 `/opt/confidential-containers/attestation-service/kds-store`，这正是 `as.verifier.snp.vcekSources` 中 `OfflineStore` 条目的读取位置。需要配合 `as.verifier.snp.nodeName`。目录布局参见 [the offline certificate cache guide](../../attestation-service/docs/amd-offline-certificate-cache.md)。 |
| as.verifier.snp.nodeName | string | `""` | 存放 kds-store 目录（`as.verifier.snp.kdsStoreHostPath`）所在的 Kubernetes 节点名称。设置了 `as.verifier.snp.kdsStoreHostPath` 时为必填；用于 PV 的 `nodeAffinity`。 |
| as.verifier.snp.vcekSources | list | `[]` | SNP verifier 的 VCEK 证书来源，按给定顺序依次尝试。为空（默认）时不输出 `snp_verifier` 配置块，AS 使用其内置默认值（KDS）。`KDS` 从 AMD 的 Key Distribution Service 获取证书，需要出网连通性。要配置 `OfflineStore` 来源，还必须提供 `as.verifier.snp.nodeName` 与 `as.verifier.snp.kdsStoreHostPath`，以便把证书存储挂载进 AS Pod。 |
| bootstrapUserKeysJob | object | `{"keygenImage":{"pullPolicy":"IfNotPresent","repository":"alpine/openssl","tag":"3.5.6"},"kubectlImage":{"pullPolicy":"IfNotPresent","repository":"quay.io/kata-containers/kubectl","tag":"20260112"},"resources":{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}}` | Bootstrap hook Job 设置（当 `secrets.useEphemeralGeneratedKeys=true` 时，负责 pre-install/pre-upgrade 的密钥生成与 post-delete 的清理）。 |
| bootstrapUserKeysJob.keygenImage.pullPolicy | string | `"IfNotPresent"` | OpenSSL `initContainer` 镜像拉取策略。 |
| bootstrapUserKeysJob.keygenImage.repository | string | `"alpine/openssl"` | 用于生成演示密钥的 OpenSSL `initContainer` 镜像仓库。 |
| bootstrapUserKeysJob.keygenImage.tag | string | `"3.5.6"` | OpenSSL `initContainer` 镜像标签。 |
| bootstrapUserKeysJob.kubectlImage.pullPolicy | string | `"IfNotPresent"` | kubectl 容器镜像拉取策略。 |
| bootstrapUserKeysJob.kubectlImage.repository | string | `"quay.io/kata-containers/kubectl"` | 用于创建或更新该 release 作用域 Secret 的 kubectl 容器镜像仓库。 |
| bootstrapUserKeysJob.kubectlImage.tag | string | `"20260112"` | kubectl 容器镜像标签。 |
| bootstrapUserKeysJob.resources | object | `{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}` | Bootstrap hook Job 的 CPU/内存 request 与 limit。 |
| dnsHostAliasWorkaround | bool | `false` | 为 `true` 时，模板会使用 Helm `lookup`，为无法解析 `*.svc.cluster.local` 的集群把 Service 的 `clusterIP` 条目写入 `hostAliases`。若首次渲染时 Service 尚不存在，请重新运行 `helm upgrade`。 |
| fullnameOverride | string | `""` | 覆盖完全限定的 release 名称（截断为 63 个字符）。 |
| ingress | object | `{"annotations":{},"className":"","enabled":false,"host":"","tls":[]}` | KBS Service 可选的 Kubernetes Ingress。 |
| ingress.annotations | object | `{}` | Ingress 注解。 |
| ingress.className | string | `""` | IngressClass 名称。 |
| ingress.enabled | bool | `false` | 为 KBS 启用 Ingress。 |
| ingress.host | string | `""` | 基于主机的路由。留空则匹配所有主机（仅通过 IP 访问）。 |
| ingress.tls | list | `[]` | TLS 配置项。 |
| kbs.affinity | object | `{}` | KBS Pod 的亲和性与反亲和性调度规则。 |
| kbs.config.admin.audience | string | `"KBS"` | bootstrap 生成的 admin token 的 JWT `audience` 声明。 |
| kbs.config.admin.issuer | string | `"TrusteeInHelm"` | bootstrap 生成的 admin token 的 JWT `issuer` 声明；必须与 `[admin.authentication.bearer_jwt]` 匹配。 |
| kbs.config.admin.role | string | `"admin"` | JWT `role` 声明，以及与之匹配的 `[admin.authorization.regex_acl]` 角色。 |
| kbs.config.attestationService.poolSize | int | `200` | KBS -> gRPC AS 客户端的连接池大小（`files/kbs-config.toml.template` 中的 `pool_size`）。 |
| kbs.config.attestationService.timeout | int | `30` | KBS -> gRPC AS 客户端的请求超时秒数（`files/kbs-config.toml.template` 中的 `timeout`）。 |
| kbs.extraEnvVars | list | `[]` | 注入 KBS 容器的额外环境变量。 |
| kbs.extraVolumeMounts | list | `[]` | KBS 容器的额外卷挂载。 |
| kbs.extraVolumes | list | `[]` | 附加到 KBS Pod 的额外卷。 |
| kbs.image.pullPolicy | string | `"Always"` | KBS 容器镜像拉取策略。 |
| kbs.image.repository | string | `"ghcr.io/confidential-containers/staged-images/kbs-grpc-as"` | KBS 容器镜像仓库。 |
| kbs.image.tag | string | `"latest"` | KBS 容器镜像标签。 |
| kbs.imagePullSecrets | list | `[]` | 私有镜像仓库可选的镜像拉取 Secret。 |
| kbs.nodeSelector | object | `{}` | KBS Pod 的节点标签选择约束。 |
| kbs.podAnnotations | object | `{}` | 额外的 Pod 注解（例如 Prometheus 抓取或服务网格集成）。 |
| kbs.podSecurityContext | object | `{}` | Pod 级别 security context 覆盖项。 |
| kbs.replicaCount | int | `1` | KBS Pod 副本数。 |
| kbs.resourceRepository | list | `[]` | KBS 资源仓库配置（透传给 KBS 配置）。 |
| kbs.resources | object | `{"limits":{"cpu":"2","memory":"2Gi"},"requests":{"cpu":"250m","memory":"256Mi"}}` | KBS 的容器 CPU/内存 request 与 limit。 |
| kbs.service.exposeLoadBalancer | bool | `false` | 为 `true` 时，额外创建一个外部 `LoadBalancer` Service（`<fullname>-kbs-lb`）。主 KBS Service（`<fullname>-kbs`）始终是内部 `ClusterIP`。 |
| kbs.service.loadBalancerAnnotations | object | `{}` | 当 `exposeLoadBalancer=true` 时，应用到可选 KBS `LoadBalancer` Service 的注解。 |
| kbs.service.port | int | `8080` | KBS 的 Service 端口；内部 `ClusterIP` Service 与可选的外部 `LoadBalancer` Service 都使用它。 |
| kbs.spire | object | `{"adminSocketPath":"/run/spire/sockets/admin.sock","attestedSelectorValue":"true","enabled":false}` | SPIRE 插件：通过既有 SPIRE 部署的 Delegated Identity API 向已通过 attestation 的 guest 分发 SPIFFE X.509-SVID。要求 KBS 镜像以 `spire-plugin` cargo feature 构建，且集群中已经运行着启用了 admin（Delegated Identity API）socket 并挂载到宿主机的 SPIRE server + agent（本 chart 自身不部署 SPIRE）。参见 `scenarios/spire.yaml` 与 `kbs/docs/plugins/spire.md`。 |
| kbs.spire.adminSocketPath | string | `"/run/spire/sockets/admin.sock"` | 在 KBS 容器内看到的 SPIRE Agent admin（Delegated Identity API）socket 路径。必须可达，例如通过 `kbs.extraVolumes` / `kbs.extraVolumeMounts` 配置的 hostPath 卷挂载（参见 `scenarios/spire.yaml`）。 |
| kbs.spire.attestedSelectorValue | string | `"true"` | 用于在每个 SVID 请求上始终添加的 `trustee:<value>` 标记 selector 的取值。 |
| kbs.spire.enabled | bool | `false` | 启用 `spire` KBS 插件（在 KBS 配置中渲染一个 `[[plugins]] name = "spire"` 配置块）。 |
| kbs.tls.enabled | bool | `false` | 在 KBS 监听器上启用原生 HTTPS。需要 `secretName`；chart 不会生成端点身份材料。 |
| kbs.tls.secretName | string | `""` | 包含 KBS HTTPS 私钥与证书链的 Secret。当 `enabled=true` 时为必填。 |
| kbs.tolerations | list | `[]` | 用于将 KBS Pod 调度到带污点节点上的容忍项。 |
| log_level | string | `"info"` | KBS、AS 与 RVPS 的容器 `RUST_LOG`（`info`、`debug`、`warn`、`error`）。 |
| nameOverride | string | `""` | 覆盖用于标签与资源名称中的 chart 名称。 |
| nodePort | object | `{"enabled":false,"port":""}` | 通过 NodePort 暴露 KBS Service。 |
| nodePort.enabled | bool | `false` | 为 KBS 启用 NodePort Service。 |
| nodePort.port | string | `""` | 固定的 NodePort 端口号；留空则从 NodePort 范围内随机分配端口。 |
| postgresql | object | `{"auth":{"database":"trustee","password":"trustee","username":"trustee"},"enabled":false,"nameOverride":"postgres","primary":{"initdb":{"scriptsConfigMap":"trustee-postgres-initdb"},"persistence":{"enabled":true,"existingClaim":"","size":"8Gi","storageClass":""},"resources":{"limits":{"cpu":"1","memory":"1Gi"},"requests":{"cpu":"250m","memory":"256Mi"}}},"service":{"ports":{"postgresql":5432}}}` | [Bitnami PostgreSQL](https://artifacthub.io/packages/helm/bitnami/postgresql) 子 chart。当需要捆绑的 Postgres 时设为 `enabled: true`（`storageBackend.postgres.mode=internal` 且需要 Postgres 存储；参见 `scenarios/postgres-backend.yaml`）。支持额外的子 chart 键（image、metrics、replication 等）；参见上游文档。 |
| postgresql.auth.database | string | `"trustee"` | 捆绑 Postgres 的数据库名（也用于 Trustee 的 `POSTGRES_URL` Secret）。 |
| postgresql.auth.password | string | `"trustee"` | 捆绑 Postgres 的密码（也用于 Trustee 的 `POSTGRES_URL` Secret）。 |
| postgresql.auth.username | string | `"trustee"` | 捆绑 Postgres 的用户名（也用于 Trustee 的 `POSTGRES_URL` Secret）。 |
| postgresql.enabled | bool | `false` | 启用 Bitnami PostgreSQL 子 chart。当 `storageBackend.postgres.mode=internal` 且需要 Postgres 存储时，必须为 `true`。 |
| postgresql.nameOverride | string | `"postgres"` | 子 chart Service 名称覆盖；release 的 Service 变为 `{Helm release}-postgres`。 |
| postgresql.primary.initdb.scriptsConfigMap | string | `"trustee-postgres-initdb"` | 通过 `templates/postgres-initdb-configmap.yaml` 与 `files/postgres-initkv.sql` 关联的 ConfigMap（除非你清楚自己在做什么，否则不要覆盖）。 |
| postgresql.primary.persistence.enabled | bool | `true` | 为捆绑的 Postgres 启用 PVC 支撑的存储。 |
| postgresql.primary.persistence.existingClaim | string | `""` | 供捆绑 Postgres 复用的已有 PVC 名称。 |
| postgresql.primary.persistence.size | string | `"8Gi"` | 自动创建的捆绑 Postgres PVC 的请求大小（例如 `8Gi`）。 |
| postgresql.primary.persistence.storageClass | string | `""` | 自动创建的捆绑 Postgres PVC 的 StorageClass；留空则使用集群默认值。 |
| postgresql.primary.resources | object | `{"limits":{"cpu":"1","memory":"1Gi"},"requests":{"cpu":"250m","memory":"256Mi"}}` | 捆绑 Postgres 的 CPU/内存 request 与 limit。 |
| postgresql.service.ports.postgresql | int | `5432` | 捆绑 Postgres 的 Service 端口（用于 `POSTGRES_URL`）。 |
| rvps.affinity | object | `{}` | RVPS Pod 的亲和性与反亲和性调度规则。 |
| rvps.extraEnvVars | list | `[]` | RVPS 容器的额外环境变量（例如 `HTTP(S)_PROXY` 与 `NO_PROXY`）。 |
| rvps.image.pullPolicy | string | `"Always"` | RVPS 容器镜像拉取策略。 |
| rvps.image.repository | string | `"ghcr.io/confidential-containers/staged-images/rvps"` | RVPS 容器镜像仓库。 |
| rvps.image.tag | string | `"latest"` | RVPS 容器镜像标签。 |
| rvps.imagePullSecrets | list | `[]` | 私有镜像仓库可选的镜像拉取 Secret。 |
| rvps.nodeSelector | object | `{}` | RVPS Pod 的节点标签选择约束。 |
| rvps.podAnnotations | object | `{}` | 额外的 Pod 注解。 |
| rvps.podSecurityContext | object | `{}` | Pod 级别 security context 覆盖项。 |
| rvps.replicaCount | int | `1` | RVPS Pod 副本数。 |
| rvps.resources | object | `{"limits":{"cpu":"1","memory":"1Gi"},"requests":{"cpu":"100m","memory":"128Mi"}}` | RVPS 的容器 CPU/内存 request 与 limit。 |
| rvps.service.loadBalancerAnnotations | object | `{}` | 内部 RVPS LoadBalancer Service 的注解。 |
| rvps.service.loadBalancerType | string | `"internal"` | 当 `rvps.service.type` 为 `LoadBalancer` 时的负载均衡器类型：`internal` 或 `public`。 |
| rvps.service.port | int | `50003` | RVPS Service 端口。 |
| rvps.service.publicLoadBalancerAnnotations | object | `{}` | 当 `loadBalancerType=public` 时，公共 RVPS LoadBalancer Service 的注解。 |
| rvps.service.type | string | `"ClusterIP"` | RVPS Service 类型（`ClusterIP` 或 `LoadBalancer`）。 |
| rvps.tolerations | list | `[]` | 用于将 RVPS Pod 调度到带污点节点上的容忍项。 |
| secrets.existingSecretName | string | `""` | 当 `useEphemeralGeneratedKeys=false` 时为必填。Secret 必须包含 `KBS_ADMIN_PRIVATE_KEY`、`KBS_ADMIN_PUBKEY`、`AS_TOKEN_SIGNING_PRIVATE_KEY` 与 `AS_TOKEN_VERIFICATION_PUBLIC_KEY_CERT_CHAIN`。可选择性包含 `KBS_ADMIN_TOKEN`（声明布局参见 `kbs/config/docker-compose/setup.sh`）。 |
| secrets.useEphemeralGeneratedKeys | bool | `true` | 为 `true` 时，pre-install/pre-upgrade hook 会把演示密钥生成到一个 release 作用域的 Secret 中；为 `false` 时，你必须预先创建 Secret 并设置 `existingSecretName`。 |
| sessionStorageType | string | `"Memory"` | KBS 协议会话存储：`Memory`、`LocalJson`、`LocalFs`、`Postgres` 或 `Redis`。留空时遵循 `storageBackend.type`。`Redis` 使用 Redis 协议，由捆绑的 Valkey 子 chart（或外部兼容 Redis 的服务）提供。 |
| storageBackend | object | `{"localFs":{"persistence":{"as":"","kbs":"","rvps":""}},"localJson":{"persistence":{"as":"","kbs":"","rvps":""}},"postgres":{"external":{"existingSecretKey":"","existingSecretName":""},"internal":{"initKvTables":true},"mode":"internal"},"redis":{"external":{"existingSecretKey":"","existingSecretName":""},"mode":"internal"},"type":"LocalFs"}` | KBS、AS 与 RVPS 统一的 KV 后端（各服务配置中使用相同的 `storage_type`）。 |
| storageBackend.localFs.persistence.as | string | `""` | AS 本地存储的 PVC 声明名称；留空则使用 `emptyDir`。 |
| storageBackend.localFs.persistence.kbs | string | `""` | KBS 本地存储的 PVC 声明名称；留空则使用 `emptyDir`。 |
| storageBackend.localFs.persistence.rvps | string | `""` | RVPS 本地存储的 PVC 声明名称；留空则使用 `emptyDir`。 |
| storageBackend.localJson.persistence.as | string | `""` | AS 本地 JSON 存储的 PVC 声明名称；留空则使用 `emptyDir`。 |
| storageBackend.localJson.persistence.kbs | string | `""` | KBS 本地 JSON 存储的 PVC 声明名称；留空则使用 `emptyDir`。 |
| storageBackend.localJson.persistence.rvps | string | `""` | RVPS 本地 JSON 存储的 PVC 声明名称；留空则使用 `emptyDir`。 |
| storageBackend.postgres.external.existingSecretKey | string | `""` | 当 `mode` 为 `external` 时为必填：Postgres URL 所在的 Secret 键名。 |
| storageBackend.postgres.external.existingSecretName | string | `""` | 当 `mode` 为 `external` 时为必填：包含 Postgres URL 的 Secret。 |
| storageBackend.postgres.internal.initKvTables | bool | `true` | 为 `true` 时，在首次初始化数据库时（通过 chart 管理的 ConfigMap）执行来自 `files/postgres-initkv.sql` 的 KV 表初始化 SQL。为 `false` 时，还应把 `postgresql.primary.initdb.scriptsConfigMap` 设为 `""`。 |
| storageBackend.postgres.mode | string | `"internal"` | Postgres 来源：`internal`（Bitnami 子 chart）或 `external`（预先创建的 Secret）。 |
| storageBackend.redis.external.existingSecretKey | string | `""` | 当 `mode` 为 `external` 时为必填：Redis URL 所在的 Secret 键名。 |
| storageBackend.redis.external.existingSecretName | string | `""` | 当 `mode` 为 `external` 时为必填：包含 Redis URL 的 Secret（例如 `redis://:password@redis.example.com:6379`）。 |
| storageBackend.redis.mode | string | `"internal"` | Redis 协议来源：`internal`（Bitnami Valkey 子 chart）或 `external`（带有 Redis URL 的预先创建的 Secret）。 |
| storageBackend.type | string | `"LocalFs"` | 后端类型：`LocalFs`、`LocalJson`、`Postgres` 或 `Memory`。当为 `Postgres`（或 `sessionStorageType` 为 `Postgres`）时，chart 会注入 `POSTGRES_URL`。只有所选类型对应的设置才会生效。 |
| valkey | object | `{"architecture":"standalone","auth":{"enabled":true,"password":"trustee"},"enabled":false,"image":{"pullPolicy":"IfNotPresent","registry":"registry-1.docker.io","repository":"bitnami/valkey","tag":"latest"},"nameOverride":"valkey","primary":{"persistence":{"enabled":false},"resources":{"limits":{"cpu":"1","memory":"512Mi"},"requests":{"cpu":"100m","memory":"128Mi"}},"service":{"ports":{"valkey":6379}}}}` | [Bitnami Valkey](https://artifacthub.io/packages/helm/bitnami/valkey) 子 chart，一个兼容 Redis 协议的存储，用于 KBS 的 `Redis` 会话后端（Valkey 为 BSD 许可；它替代了许可证已不再被 OSI 认可的 Redis）。当需要捆绑的存储时设为 `enabled: true`（`storageBackend.redis.mode=internal` 且 `sessionStorageType` 或 `storageBackend.type` 为 `Redis`；参见 `scenarios/valkey-sessions.yaml`）。支持额外的子 chart 键（metrics、replication、TLS 等）；参见上游文档。 |
| valkey.architecture | string | `"standalone"` | 单个 Valkey 主节点；会话不需要副本。如需高可用，请设置 `replication` 以及 `replica.*` 键（参见上游文档）。 |
| valkey.auth.enabled | bool | `true` | 为捆绑的 Valkey 要求密码（也用于 Trustee 的 `REDIS_URL` Secret）。 |
| valkey.auth.password | string | `"trustee"` | 捆绑 Valkey 的密码（也用于 Trustee 的 `REDIS_URL` Secret）。 |
| valkey.enabled | bool | `false` | 启用 Bitnami Valkey 子 chart。当 `storageBackend.redis.mode=internal` 且需要 Redis 存储时，必须为 `true`。 |
| valkey.image | object | `{"pullPolicy":"IfNotPresent","registry":"registry-1.docker.io","repository":"bitnami/valkey","tag":"latest"}` | 捆绑 Valkey 的容器镜像。默认来自 `docker.io`，该仓库对匿名拉取有速率限制；请把 `registry` / `repository` 指向私有镜像仓库以避免拉取失败（子 chart 也同样采纳 chart 级别的 `global.imageRegistry`）。 |
| valkey.image.pullPolicy | string | `"IfNotPresent"` | Valkey 镜像拉取策略。 |
| valkey.image.registry | string | `"registry-1.docker.io"` | Valkey 镜像仓库；可覆盖为镜像站以避免 docker.io 的速率限制。 |
| valkey.image.repository | string | `"bitnami/valkey"` | Valkey 镜像仓库路径。 |
| valkey.image.tag | string | `"latest"` | Valkey 镜像标签。 |
| valkey.nameOverride | string | `"valkey"` | 子 chart 名称覆盖；主 Service 变为 `{Helm release}-valkey-primary`。 |
| valkey.primary.persistence.enabled | bool | `false` | KBS 会话是短期的，因此捆绑的 Valkey 默认不带 PVC；设为 `true`（并可选设置 `storageClass` / `size`）可在 Pod 重启后保留会话。 |
| valkey.primary.resources | object | `{"limits":{"cpu":"1","memory":"512Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}` | 捆绑 Valkey 主节点的 CPU/内存 request 与 limit。 |
| valkey.primary.service.ports.valkey | int | `6379` | 捆绑 Valkey 的 Service 端口（用于 `REDIS_URL`）。 |

## 端到端测试

在外部额外准备一个 **kind** 集群，把 Trustee 镜像预加载进该集群，让 `kubectl` 指向它，然后从仓库根目录执行：

```bash
kind create cluster --name kind --wait 5m
make -C deployment/helm-chart load-e2e-images-into-kind
make test-helm-e2e
```

`make e2e-test` 假定 **KBS / AS / RVPS** 镜像已经加载到集群中，并使用 [scenarios/e2e-local-images.yaml](./scenarios/e2e-local-images.yaml) 进行部署。Makefile 会在安装时注入镜像仓库、标签以及 `pullPolicy: Never`，这样本地运行与 CI 可以使用不同的预加载镜像名，而无需修改场景文件。

- **本地预加载镜像**：`trustee-e2e/*:e2e`，`pullPolicy: Never`（而非 GHCR 的 `:latest`）
- **CI 镜像**：复用 `workflow-call-build-docker-e2e-materials.yml` 中的 `docker-e2e-images-linux-amd64`，作为 `ghcr.io/confidential-containers/staged-images/*:latest`
- **存储**：捆绑的 Postgres KV 后端（`storageBackend.type: Postgres`）
- **KBS 会话**：捆绑的 Valkey（`sessionStorageType: Redis`）
- **KBS 端点**：使用一次性、仅用于测试的证书的原生 HTTPS

步骤：

1. **`helm-dependency-build`**
2. **`helm-lint`** — 校验默认 HTTP、e2e 与原生 HTTPS 三种配置
3. **`prepare-e2e-tls`** — 创建一个测试用 Kubernetes TLS Secret
4. **`deploy`**
5. **`test-client`** — [e2e/test.sh](./e2e/test.sh) 校验证书并通过 HTTPS 调用 KBS
6. **`dump-logs`** — 仅在失败时执行：在命名空间仍然存在时，输出 Pod 状态、事件、describe 以及容器日志（当前 + 上一次）
7. **`undeploy`** — 总是会尝试执行，即使前面发生失败

单独调试：

```bash
make -C deployment/helm-chart load-e2e-images-into-kind
make -C deployment/helm-chart helm-lint
make -C deployment/helm-chart prepare-e2e-tls
make -C deployment/helm-chart deploy
make -C deployment/helm-chart test-client
make -C deployment/helm-chart dump-logs
make -C deployment/helm-chart undeploy
```

可选变量：`E2E_IMAGE_PREFIX`、`E2E_IMAGE_TAG`、`KBS_CLIENT`。

CI（`test-e2e-kbs.yml`）复用 Docker Compose e2e 镜像构建工作流的产物：它会把预构建好的 Trustee 镜像加载到 kind 中，并使用预构建的 `kbs-client` 二进制，然后再运行 Helm e2e。

单独执行 `make test-client` 不会构建镜像，也不会卸载部署。

### SPIRE 插件 e2e 测试

另有一个 e2e 测试用于验证 **`spire`** KBS 插件。与主 e2e 测试不同，它在用 [scenarios/spire.yaml](./scenarios/spire.yaml) 部署 Trustee 之前，还会先安装一套一次性的 SPIRE server + agent + `spiffe-csi`（通过上游的 `spiffe/helm-charts-hardened` charts）：

```bash
kind create cluster --name kind-spire --wait 5m
make -C deployment/helm-chart load-e2e-images-into-kind KIND_CLUSTER=kind-spire
make -C deployment/helm-chart e2e-test-spire
```

单独调试：

```bash
make -C deployment/helm-chart load-e2e-images-into-kind KIND_CLUSTER=kind-spire
make -C deployment/helm-chart install-spire-stack
make -C deployment/helm-chart deploy-spire
make -C deployment/helm-chart test-client-spire
make -C deployment/helm-chart dump-logs
make -C deployment/helm-chart undeploy
```

**`test-client-spire`** — [e2e/spire-test.sh](./e2e/spire-test.sh) 注册 SPIRE 条目，以 sample TEE 身份完成 attestation，并通过插件的 `x509-svid` / `x509-bundles` 资源获取 X.509-SVID 与信任包（trust bundle）。

CI 在 `test-e2e-kbs.yml` 中以独立的 `helm-trustee-e2e-spire` job 运行它，复用与主 e2e job 相同的预构建镜像与 `kbs-client` 产物。

## 开发说明

1. 除非你正在更新对应的 Helm 模板逻辑，否则不要修改 [files](./files/) 下的文件。服务配置由 `*.template` 文件渲染而来；Postgres KV 初始化 SQL 位于 `files/postgres-initkv.sql`。
2. 修改 `Chart.yaml` 中的依赖项后，请运行 `helm dependency update ./deployment/helm-chart`，并在你的打包工作流会内联（vendor）子 chart 时，一并提交 `Chart.lock` 与 `charts/`。
3. 修改 `values.yaml` 的注释或键之后，请从 [README.md.gotmpl](./README.md.gotmpl) 重新生成本 README：`helm-docs -c .`（在本目录中运行）。
