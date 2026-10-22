# Nebula CA 插件

> 说明：本文件是 `kbs/docs/plugins/nebula_ca.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../custom-文档清单.md)。

[Nebula](https://github.com/slackhq/nebula) 是一个开源项目，它提供了用于创建三层加密型 Nebula 覆盖网络（Layer 3 Encrypted Nebula Overlay Network，ENON）的工具。每个 Nebula 发行版提供两个二进制文件。
- nebula：用于创建节点（Lighthouse 或普通节点），并加入某个 Lighthouse 的 ENON。
- nebula-cert：用于生成密钥、证书、CA，以及对节点证书进行签名的可执行程序。

本插件会调用 `nebula-cert` 二进制，为希望加入某个 ENON 的节点（例如 CoCo POD 或机密虚拟机）提供其部分 CA 功能。

每个 ENON 必须至少有一个 Lighthouse。Lighthouse 是一个拥有静态 IP 地址的节点，它标识该 ENON 并协助节点发现。

## 安装配置

1. 构建 KBS 时启用 cargo 特性 `nebula-ca-plugin`，并把 `nebula-cert` 二进制安装到 KBS 镜像中。

```bash
docker compose build --build-arg NEBULA_CA_PLUGIN=true
``` 

2. 配置 `nebula-ca` 插件。对于简单场景，插件的默认配置就足够了，只需把下面几行加入到 [KBS 配置](#kbs/config/docker-compose/kbs-config.toml)中。更复杂的场景请参阅 [config.md](#kbs/docs/config.md)。

```toml
[[plugins]]
name = "nebula-ca"
```

3. 启动 trustee

```bash
docker compose up
```

## 运行时服务

下面各节描述了本插件支持的全部运行时服务。

### credential 服务

为节点创建一份凭证，以便其加入某个 ENON。

仅支持 `GET` 请求，例如 `GET /kbs/v0/nebula-ca/credential?name=podA&ip=10.9.8.7/21`。

该请求通过 URL 查询字符串接收参数。下表描述了全部支持的参数。注意 `name` 和 `ip` 是必填项。

| 属性               | 类型   | 必填 | 说明                     | 默认值 | 示例                                   |
|---------------------|--------|----------|-------------------------|---------|-------------------------------------------|
| `name`              | String | 是      | 证书名称，通常是主机名或 Pod 名称 |         | `credential?name=podA&ip=10.9.8.7/21` |
| `ip`                | String | 是      | 要分配给该证书的 IPv4 地址及网络，使用 CIDR 表示法 |         | `credential?name=podA&ip=10.9.8.7/21` |
| `duration`          | String | 否       | 证书的有效期。 | 比签名证书到期时间提前 1 秒。有效的时间单位为：<小时>"h"<分钟>"m"<秒>"s" | `credential?name=podA&ip=10.9.8.7/21&duration=8760h0m0s` |
| `groups`            | String | 否       | 以逗号分隔的组列表 |         | `credential?name=podA&ip=10.9.8.7/21&groups=ssh,server` |
| `subnets`           | String | 否       | 以逗号分隔的 IPv4 地址及网络列表，使用 CIDR 表示法。表示该证书可以为之提供服务的子网。 |         | `credential?name=podA&ip=10.9.8.7/21&subnets=10.9.7.7/21,10.9.8.7/21` |

只有当节点通过远程证明时，请求才会被处理，否则会返回错误。借此，ENON 中预期只会包含已通过远程证明的节点。

请求被处理后，会以 JSON 格式返回如下结构。

```rust
struct CredentialServiceOut {
    node_crt: Vec<u8>,  // 生成的自签名证书
    node_key: Vec<u8>,  // 生成的密钥
    ca_crt: Vec<u8>,    // CA 证书
}
```

目前，该服务仅提供基础功能。
- 它是无状态的。一旦某份被请求的凭证被返回，它就会被删除。
- 它不支持 [CA 轮换](https://nebula.defined.net/docs/guides/rotating-certificate-authority/)。
- 它不支持运行时远程证明。如果同一个 POD 稍后再次请求另一份凭证，该 POD 相对初始状态所做的变更不会被再次度量。理想情况下，POD 应确保证书在工作负载结束之前不会过期。
- 它不掌握任何关于 Lighthouse 的信息，因此无法检查请求中提供的 IP 地址与 Lighthouse 的 IP 地址是否位于同一网络中。
