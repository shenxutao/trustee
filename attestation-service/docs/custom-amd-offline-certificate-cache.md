# AMD VCEK 离线存储指南

> 说明：本文件是 `attestation-service/docs/amd-offline-certificate-cache.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

本文档描述如何把 VCEK（Versioned Chip Endorsement Key，版本化芯片背书密钥）预加载到 trustee 环境中，
以便在不连接 AMD KDS 的情况下完成正常的证明。本指南主要面向隔离网络（air-gapped）环境。

> [!Note]
> 本指南同时涵盖 [Docker Compose](https://confidentialcontainers.org/docs/attestation/installation/docker/)
> 与 [Helm](https://confidentialcontainers.org/docs/attestation/installation/helm/) 两种部署方式。

## 在证明服务中启用离线 VCEK 存储

### 1. 设置 VCEK 来源配置项

更新证明配置文件，使其使用预置的 vcek 存储：

`kbs/config/docker-compose/as-config.json`：

```json
{
    // ... 其他字段 ...
    "verifier_config": {
        "snp_verifier": {
          // 配置要依次尝试的 VCEK 来源。默认为 [KDS]。
            "vcek_sources": [
                {
                  "type": "OfflineStore",
                },
                // 可选地添加一个回退到 KDS 的来源。如果 KDS 不可达则不要添加。
                //{
                //  "type": "KDS",
                //}
            ]
        }
    }
}
```

一旦指定了 `OfflineStore` 配置，Trustee 就会检查所配置的目录中的 VCEK 值。

给定一份证明报告，trustee 会首先查找：
```
# 优先（基于 TCB 的文件名，若存在）：
/opt/confidential-containers/attestation-service/kds-store/vcek/{hwid}/{tcb_prefix}_vcek.der
```

如果该查找失败，则会查找旧有的扁平格式：
```
# 回退（旧有扁平布局）：
/opt/confidential-containers/attestation-service/kds-store/vcek/{hwid}/vcek.der
```

其中：
- `{hwid}` 是 AMD EPYC 服务器的硬件 ID，为小写十六进制
- `{tcb_prefix}` 是基于 TCB 参数的文件名前缀，格式为：
  - 对于非 Turin 处理器：`bl{BL}_tee{TEE}_snp{SNP}_ucode{UCODE}`
  - 对于 Turin 处理器：在上述基础上追加 `_fmc{FMC}`

  每个参数都补零至 2 位（例如 `bl02_tee00_snp06_ucode21`）

### 2. 创建并填充 VCEK 目录

创建一个 `vcek` 目录，并按硬件 ID 填充为以下结构之一。推荐使用带 TCB 前缀的文件名，
以便为同一台主机加载多个 VCEK。

```
vcek/
├── <Hex hardware ID>/ （ID 必须为小写）
│   ├── vcek.der                          # 扁平布局（回退）
│   └── <TCB prefix>_vcek.der             # 可选，存在时优先使用
├── <Hex hardware ID>/
│   └── vcek.der
└── <Hex hardware ID>/
    └── <TCB prefix>_vcek.der
```

`<TCB prefix>` 由证书的 TCB 参数派生，格式为 `bl{BL}_tee{TEE}_snp{SNP}_ucode{UCODE}`
（对于 Turin 处理器追加 `_fmc{FMC}`）。例如：`bl02_tee00_snp06_ucode21_vcek.der`

KBS 所服务的每一台物理 AMD EPYC 服务器都需要一份唯一的证书。
服务器的硬件 ID 与证书 URL 可以使用 [snphost 工具](https://github.com/virtee/snphost)获取。

在每台 AMD SNP 主机上，以 root/admin 权限运行：
```
sudo snphost show vcek-url
```

你会得到如下形式的 URL：
```
https://kdsintf.amd.com/vcek/{version}/{machine}/{product_name}/{hwid}?{params}
```

你可以在浏览器中粘贴该 URL 进行下载，也可以在 SNP 主机本身上运行以下命令
（它会从本地固件派生出该 URL）。需要能够访问 AMD 的 KDS 网络。
```
sudo snphost fetch vek der .
```

如果要从另一台机器（例如构建机）获取，请显式传入该 URL，
这样证书才会与源主机的固件匹配，而不是本机的固件：
```
sudo snphost fetch vek der . "<vcek-url>"
```

> [!Note]
> 较老版本的 `snphost` 使用 `snphost fetch vcek` 而不是 `vek`。不过建议升级到最新版本的 `snphost`。

> [!IMPORTANT]
> - 请注意，VCEK URL 与机器上的 AMD 硬件固件相关。如果固件被更新，VCEK URL 会随之改变。
> 关于 VCEK URL 格式的更多信息，参见 [AMD VCEK 文档](https://docs.amd.com/v/u/en-US/57230)。

### 3. 把 `vcek` 目录安装到 trustee 部署中

#### Helm 部署

1. 在 Kubernetes 节点上把 VCEK 证书暂存到：
   ```
   /opt/confidential-containers/attestation-service/kds-store/vcek/{hwid}/{tcb_prefix}_vcek.der  （优先）
   ```
   或
   ```
   /opt/confidential-containers/attestation-service/kds-store/vcek/{hwid}/vcek.der  （回退）
   ```

2. 在启用离线 VCEK 存储的前提下安装 Trustee：
   ```bash
   helm install trustee ./deployment/helm-chart \
     --namespace coco-trustee --create-namespace \
     --set as.verifier.snp.kdsStoreHostPath=/opt/confidential-containers/attestation-service/kds-store \
     --set as.verifier.snp.nodeName=<node-name> \
     --set 'as.verifier.snp.vcekSources[0].type=OfflineStore'
   ```

   将 `<node-name>` 替换为存放 VCEK 证书的 Kubernetes 节点主机名（可用 `kubectl get nodes` 查到）。

   `as.verifier.snp.vcekSources` 默认为 `[]`，此时会省略 `snp_verifier` 配置块，
   让 AS 使用其内置默认值（KDS）。必须显式请求一个 `OfflineStore` 来源。
   它需要 `kdsStoreHostPath` 和 `nodeName`，因为正是这两个参数把证书存储挂载进 AS Pod。
   若要保留 AMD KDS 作为本地缺失证书的回退：
   ```bash
   --set 'as.verifier.snp.vcekSources[0].type=OfflineStore' \
   --set 'as.verifier.snp.vcekSources[1].type=KDS'
   ```

#### Docker Compose 部署

- **在运行中的 trustee 部署里安装**

使用 docker 命令把你的 `vcek` 目录复制到已配置的目录中：
```
sudo docker exec trustee-as-1 mkdir -p /opt/confidential-containers/attestation-service/kds-store/
sudo docker cp ./vcek/ trustee-as-1:/opt/confidential-containers/attestation-service/kds-store/vcek/
```

- **挂载共享目录**

你也可以通过更新 `docker-compose.yml`、添加一个指定的目录映射，
把宿主机上的共享目录挂载进容器：

```yaml
  as:
    ... <existing configuration>
    volumes:
    ... <existing volumes>
    - ./vcek:/opt/confidential-containers/attestation-service/kds-store/vcek:rw
```

### 示例：

对于你希望 trustee 服务的若干台 AMD EPYC 服务器：
```
ssh privileged-user@epyc-host "sudo snphost show vcek-url" >> urls.txt
```

在一台可以访问 AMD KDS 网络的系统上：
```bash
mkdir vcek
# qval <tcb-key> <query-string>：从查询串中提取某个 TCB 键的 SPL 值
qval() { grep -oP "$1SPL=\K[0-9]+" <<<"$2"; }

# url_hwid <vcek-url>：从 URL 路径派生出小写硬件 ID
url_hwid() { tr '[:upper:]' '[:lower:]' <<<"${1##*/}" | cut -d'?' -f1; }

while read -r url; do
  hwid=$(url_hwid "$url")
  query="${url#*\?}"  # 如果没有 '?'，则得到整个 URL；下面会生成扁平布局

  tcb_prefix=""
  for key in bl tee snp ucode fmc; do
    val=$(qval "$key" "$query") || continue
    tcb_prefix+="${tcb_prefix:+_}${key}$(printf '%02d' "$((10#$val))")"
  done

  mkdir -p "vcek/$hwid"
  # snphost 会把证书写入给定目录下的 vcek.der；重命名为带 TCB 前缀的文件名。
  # 如果没有解析出 TCB 参数，则保留扁平文件名。
  (
    cd "vcek/$hwid" || exit 1
    sudo snphost fetch vek der . "$url"
    [ -n "$tcb_prefix" ] && mv vcek.der "${tcb_prefix}_vcek.der"
  )
done < urls.txt

scp -r vcek user@target-host:/path/to/trustee
```

在隔离网络的 trustee attestation-service 主机上：
```bash
# 更新 as-config.json 以启用离线 VCEK 存储
cd /path/to/trustee
vi kbs/config/docker-compose/as-config.json
# 把 verifier_config 小节更新为：
#     "verifier_config": {
#        "snp_verifier": {
#            "vcek_sources": [
#              {
#                "type": "OfflineStore",
#              }
#            ]
#        }
#    }

# 启动 trustee
docker compose up -d

# 把 vcek 存储复制进运行中的证明服务容器
sudo docker exec trustee-as-1 mkdir -p /opt/confidential-containers/attestation-service/kds-store/
sudo docker cp ./vcek/ trustee-as-1:/opt/confidential-containers/attestation-service/kds-store/vcek/

# 除了 docker copy 之外，另一种做法是在 docker-compose.yml 的 as.volumes 小节
# 添加如下共享挂载：
# - ./vcek:/opt/confidential-containers/attestation-service/kds-store/vcek:rw
```

## 限制

在以下情况下必须更新/重建 VCEK 存储：
- 向被服务集群中新增了 AMD EPYC 主机
- AMD EPYC 主机的固件组件被更新
- VCEK 证书被 CA 吊销

## 故障排查

开启 debug 日志，并在日志中检索 `vcek` 关键字。

**Helm：**
```bash
helm upgrade trustee ./deployment/helm-chart -n coco-trustee --set log_level=debug
kubectl logs -n coco-trustee deploy/trustee-as -c as | grep -i vcek
```

**Docker Compose：**
```bash
echo "RUST_LOG=debug" > debug.env
docker compose --env-file debug.env up -d
docker logs trustee-as-1 | grep -i vcek
```
