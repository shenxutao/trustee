# HashiCorp Vault 与 OpenBao KV secrets engine 资源后端

> 说明：本文件是 `kbs/docs/vault_kv.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

[HashiCorp Vault](https://developer.hashicorp.com/vault) 与 [OpenBao](https://www.openbao.org/)
是密钥管理工具，用于安全地存储并严格控制对令牌、口令、证书、加密密钥及其他机密的访问。
本后端把 KV v1（Key-Value 版本 1）secret engine 集成进来，
作为密钥代理服务（KBS）的存储后端。

该后端对 Vault 与 OpenBao 都适用。KV v1 的 API 与路径完全相同，
因此下文描述的 KBS 配置、路径映射与令牌策略对两者都适用。
实践中唯一的区别是服务端设置所用的 CLI：HashiCorp Vault 使用 `vault` CLI，
OpenBao 使用 `openbao` CLI（例如 `openbao secrets enable -version=1 -path=kv kv`）。

KV v1 后端允许 KBS 从一个集中式的 Vault 或 OpenBao 实例中存储和检索机密资源
（secret、密钥、证书等），并获得企业级的安全特性，包括访问控制、审计日志与静态加密。

## 特性

- **读/写操作**：完整支持从 Vault KV v1 engine 存储与检索 secret
- **HTTPS 支持**：通过 TLS/SSL 与 Vault 服务进行安全通信
- **自定义 CA 证书**：支持企业级与自签名的证书颁发机构
- **令牌认证**：使用 Vault 令牌进行安全认证
- **路径映射**：自动把 KBS 资源描述符映射到 Vault 路径

## 设置

### 1. 启用 Vault 特性

`vault` 后端默认已启用。

如需禁用：

```bash
cd kbs
make VAULT=false
```

### 2. 配置 Vault 或 OpenBao 访问

确保你的 Vault 或 OpenBao 实例正在运行且可访问。该后端需要：

- 一个运行中的、已启用 KV v1 engine 的 Vault 或 OpenBao 服务
- 一个具备相应权限的有效令牌
- 从 KBS 到该服务的网络连通性

以下示例使用 `vault` CLI。如果你使用 OpenBao，
请在每条命令中把 `vault` 替换为 `openbao`；参数与行为都是相同的。

#### Vault / OpenBao 配置示例

```bash
# 启用 KV v1 engine（如果尚未启用）
vault secrets enable -version=1 -path=kv kv

# 为 KBS 访问创建一条策略
vault policy write kbs-policy - <<EOF
path "kv/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

# 为 KBS 创建一个令牌
vault token create -policy=kbs-policy
```

### 3. 配置 KBS

把 Vault 配置加入到你的 KBS 配置文件（例如 `kbs-config.toml`）中：

```toml
[[plugins]]
name = "resource"
storage_backend_type = "Vault"
vault_url = "https://vault.example.com:8200"
token = "hvs.your-vault-token-here"
mount_path = "kv"                              # 可选，默认为 "secret"
verify_ssl = true                              # 可选，默认为 false
ca_certs = ["/path/to/ca-bundle.pem"]          # 可选，自定义 CA 证书
```

### 4. 启动 Trustee

```bash
# 使用 docker-compose
docker compose up

# 或者直接启动
../target/release/kbs --config-file /path/to/kbs-config.toml
```

## 配置项

| 属性      | 类型           | 是否必需 | 说明                                              | 默认值     |
|---------------|----------------|----------|----------------------------------------------------------|-------------|
| `vault_url`   | String         | 是      | Vault 或 OpenBao 服务 URL（HTTP 或 HTTPS）              | -           |
| `token`       | String         | 是      | Vault 或 OpenBao 认证令牌                    | -           |
| `mount_path`  | String         | 否       | KV v1 挂载路径                                         | `"secret"`  |
| `verify_ssl`  | Boolean        | 否       | 启用/禁用 SSL 证书校验              | `false`     |
| `ca_certs`    | Array[String]  | 否       | 自定义 CA 证书文件的路径                     | `None`      |

### HTTPS 配置

该后端支持通过全面的 TLS 配置项与服务器进行安全的 HTTPS 通信：

#### 基础 HTTPS（默认）

```toml
[[plugins]]
name = "resource"
storage_backend_type = "Vault"
vault_url = "https://vault.example.com:8200"
token = "hvs.your-vault-token-here"
verify_ssl = true
```

#### 自定义 CA 证书

对于使用自定义证书颁发机构的企业环境：

```toml
[[plugins]]
name = "resource"
storage_backend_type = "Vault"
vault_url = "https://vault.mycompany.com:8200"
token = "hvs.your-vault-token-here"
verify_ssl = true
ca_certs = [
    "/etc/ssl/certs/company-ca.pem",
    "/opt/vault/tls/vault-ca.pem"
]
```

#### 开发/测试（禁用 SSL 校验）

```toml
[[plugins]]
name = "resource"
storage_backend_type = "Vault"
vault_url = "http://vault-dev.mycompany.com:8200"
token = "hvs.your-vault-token-here"
verify_ssl = false
```

## 路径映射

该后端按以下格式自动把 KBS 资源描述符映射到 Vault/OpenBao 路径：

```bash
{repository_name}/{resource_type}/{resource_tag}
```

secret 的值存放在键 `data` 之下。


### 示例

| KBS 资源请求 | Vault 路径 |
|---------------------|------------|
| Repository：`default`，Type：`key`，Tag：`encryption-key-1` | `default/key/encryption-key-1` |
| Repository：`app1`，Type：`cert`，Tag：`tls-cert` | `app1/cert/tls-cert` |
| Repository：`prod`，Type：`secret`，Tag：`db-password` | `prod/secret/db-password` |

实际的 secret 值存放在给定路径下键 `data` 之下。例如，
该 secret 就是 `prod/secret/db-password` 路径下键 `data` 的值。

## 令牌要求

KBS 所使用的令牌必须对已配置的挂载路径（无论服务端是 Vault 还是 OpenBao）具备以下能力：

```hcl
path "{mount_path}/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

### Vault 策略示例

#### 最小策略（位于默认 "secret" 挂载点的 KV v1）

```hcl
path "secret/*" {
  capabilities = ["create", "read", "update", "delete"]
}
```

#### 自定义挂载路径

```hcl
path "kv/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

#### 受限访问（只读）

```hcl
path "secret/*" {
  capabilities = ["read"]
}
```

## 故障排查

### Debug 日志

开启 debug 日志以排查问题：

```bash
RUST_LOG=debug ./target/release/kbs --config-file /path/to/kbs-config.toml
```

## 测试

该后端包含使用
[OpenBao](https://openbao.org/) 的集成测试，
OpenBao 是由 Linux 基金会维护的、与 API 兼容的 HashiCorp Vault 开源分支。
测试的 Makefile 会自动下载 OpenBao 二进制并启动一个本地服务。

在不设置 SSL 的情况下运行测试：

```sh
cd kbs/test
make test-vault-nossl
make stop-vault
```

在设置 SSL 的情况下运行测试：

```sh
cd kbs/test
make test-vault-ssl
make stop-vault-ssl
```

## 限制

- **KV v1 支持**：目前仅支持 KV v1 engine
- **认证方式**：仅支持令牌认证
- **挂载点**：每个实例只能有一个挂载点
- **二进制数据**：所有数据都以 UTF-8 字符串形式存储

## 相关文档

- [HashiCorp Vault 文档](https://developer.hashicorp.com/vault/docs)
- [Vault KV v1 Secret Engine](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v1)
- [OpenBao 文档](https://openbao.org/docs)
- [OpenBao KV v1 API](https://openbao.org/api-docs/secret/kv/kv-v1/)
