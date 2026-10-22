# KBS 集群

> 说明：本文件是 `kbs/docs/cluster.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

KBS 提供了一个由 `docker-compose` 定义的简单集群，包含自身、[证明服务](../../attestation-service/)、[参考值供给服务](../../rvps/)以及 [CoCo Keyprovider](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent/coco_keyprovider)。

用户只需非常简单的命令即可：
- 启动 KBS 服务。
- 加密镜像。

## 架构

<div align=center>

![](./pictures/cluster.svg)

</div>

## 启动

运行该集群
```bash
git clone https://github.com/confidential-containers/trustee.git
cd trustee
docker compose up -d
```

`setup` 容器会自动初始化 `kbs/config/docker-compose/` 下的文件：

- `private.key` / `public.pub`：供 KBS 管理 API 使用的管理员 JWT 签名与验签密钥。
- `admin-token`：供本地开发使用的长期有效的管理员 Bearer 令牌。
- `ca-cert.pem`、`token.key`、`token-cert-chain.pem` 及相关文件：
  AS 与 KBS 使用的证明令牌签名与信任链材料。
  - `token.key` + `token-cert-chain.pem` —— 配置为 AS 的 `signer`，用于签发 JWT。
  - `ca-cert.pem` —— 被列在 KBS `[attestation_token].trusted_certs_paths` 中的根 CA，
    以便 KBS 能够校验令牌头部中的 `x5c` 证书链。

  这些文件如何配合使用，请参见
  [证明令牌校验](./attestation_token_verification.md#docker-compose-cluster)。

  默认情况下 `ca-cert.pem` 有效期为 3650 天（10 年），`token-cert.pem` 为
  90 天。如果你需要不同的有效期，可在 `docker compose up` 之前覆盖它们：

  ```bash
  CA_CERT_DAYS=1825 TOKEN_CERT_DAYS=30 docker compose up -d
  ```

将生成的管理员令牌用于 `kbs-client`：

```bash
kbs-client config \
  --url http://127.0.0.1:8080 \
  --admin-token-file kbs/config/docker-compose/admin-token \
  set-resource-policy --allow-all
```

如果省略 `--admin-token-file`，`kbs-client` 会按以下顺序解析管理员 Bearer 令牌：

1. `kbs/config/docker-compose/admin-token` —— 相对于**当前工作目录**（通常在 `cd trustee` 之后就是仓库根目录）。该文件由 `setup` 服务在 docker-compose trustee 启动时创建。
2. `$HOME/.trustee/admin-token` —— 仅当 `$HOME` 已设置且该文件存在时使用。
3. 不使用令牌 —— 管理请求将不带认证信息发出。这只在 KBS `admin.authorization_mode` 为 `InsecureAllowAll` 时可行。`docker-compose` 集群默认为 `AuthenticatedAuthorization`，因此你需要来自第 1 步或第 2 步的令牌（或显式传入 `--admin-token-file`）。

注意，默认情况下 KBS 集群会拒绝 sample 证据。
如果你使用 sample 证据进行测试，就需要
设置更宽松的资源策略。

随后 KBS 集群便启动完成。

使用 `skopeo` 加密镜像
```bash
# edit ocicrypt.conf
tee > ocicrypt.conf <<EOF
{
    "key-providers": {
        "attestation-agent": {
            "grpc": "127.0.0.1:50000"
        }
    }
}
EOF

# encrypt the image
OCICRYPT_KEYPROVIDER_CONFIG=ocicrypt.conf skopeo copy --insecure-policy --encryption-key provider:attestation-agent docker://busybox oci:busybox_encrypted
```

镜像将被加密，后台发生的事情包括：
- `CoCo Keyprovider` 生成一个随机的 KEK 和一个 key id，然后使用该 KEK 加密镜像。
- `CoCo Keyprovider` 将该 KEK 连同 key id 注册到 KBS。

如果使用同一个 KBS 进行密钥代理，则该镜像可以被解密。
