# Provisioner 插件

> 说明：本文件是 `kbs/src/plugins/implementations/provisioner/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../../custom-文档清单.md)。

provisioner 插件用于为机密虚拟机自动生成 LUKS 密钥并创建资源。
它被设计为与一个 provisioner operator（或 hook sidecar）配合工作：
在虚拟机启动之前，由该 operator 调用本插件，以获取把虚拟机与其加密密钥绑定所需的取值。

## 流程

1. operator 发送 `POST /kbs/v0/provisioner/provision`，并携带虚拟机身份。
2. 插件从该虚拟机身份派生出一个确定性的 UUID，并生成一个随机的 LUKS 加密密钥。
3. 该密钥被持久化到 KBS 存储后端中，以便 guest 在证明之后能够经本插件取回它。
4. 插件返回一个 JSON 响应，其中包含 operator 必须注入到虚拟机中的取值。

在启动时，guest 向 KBS 完成证明，并通过 `/kbs/v0/provisioner/...` 路径获取该密钥。
在 guest 内部，`systemd-repart` 使用该密钥加密磁盘，
而 `systemd-cryptsetup` 在后续启动时使用它解密磁盘。

## Provision 响应

返回给 operator 的 JSON 响应：

```json
{
  "uuid": "c007d2cc-5f14-41e5-a202-8d15f6f68607",
  "resource_path": "default/c007d2cc-5f14-41e5-a202-8d15f6f68607/root"
}
```

### 字段
- **`uuid`**：虚拟机的确定性标识符，operator 用它来构造 initdata 以及执行 deprovision 操作。
- **`resource_path`**：存放该 LUKS 密钥的 KBS 资源路径。operator 会把它嵌入 guest 的 initdata 中，
  这样 guest 就知道在证明之后该从哪里获取自己的密钥。

## 资源格式

存放在 KBS 存储后端中的资源遵循
`trustee-attester` 所期望的
[confdata](https://gitlab.com/berrange/cvminjector#confidential-data-format)
TOML 格式：

```toml
version = "0.1.0"

[data]
"io.cryptsetup.key.text.root" = "<random LUKS key>"
```

guest 在证明之后从 `/kbs/v0/provisioner/default/{uuid}/root` 获取该资源，
并解析 `io.cryptsetup.key.text.root` 字段以获得 LUKS 口令。

## Deprovision（撤销供给）

要移除一个已供给的资源：

```
DELETE /kbs/v0/provisioner/provision/{uuid}
```

## 认证

本插件根据调用方使用两条认证路径：

- **POST/DELETE**（operator）：`validate_auth` 返回 `true`，
  走 KBS 的 admin 认证。operator 属于基础设施，而不是 TEE。
- **GET**（guest 虚拟机）：`validate_auth` 返回 `false`，
  要求一个由证明服务校验并依据 KBS 资源策略评估的 TEE 证明令牌。
  响应会使用 TEE 公钥进行 JWE 加密。
