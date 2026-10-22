# KBS 管理模块

> 说明：本文件是 `kbs/docs/admin.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

`kbs/src/admin` 模块用于保护 KBS 的管理 API。它支持三种模式：

- `InsecureAllowAll`
- `DenyAll`
- `AuthenticatedAuthorization(Authentication, Authorization)`

在 `AuthenticatedAuthorization` 模式下，KBS 会先校验管理员令牌，然后再应用端点授权规则。

> [!NOTE]
> 实现一个完整的认证与授权模块并不是 KBS 的目标。因此管理模块这部分应当尽可能保持简单。
> 如果需要复杂的访问控制，请在 KBS 前面使用一个带外（out-of-band）网关。

## 管理模式与配置

### 1）InsecureAllowAll（全部放行）

该模式允许所有管理请求。它适用于本地开发，或者在 KBS 前面已有带外 authN + authZ 服务的生产环境。

```toml
[admin]
authorization_mode = "InsecureAllowAll"
```

### 2）DenyAll（全部拒绝）

该模式拒绝所有管理请求。适用于应当禁用管理 API 的场景。

```toml
[admin]
authorization_mode = "DenyAll"
```

### 3）AuthenticatedAuthorization（已认证授权）

该模式启用真正的管理员认证与授权。

- `authentication = bearer_jwt` 校验 `Authorization: Bearer <JWT>`
- JWT **必须**包含 `role` claim
- `authorization = regex_acl` 依据 `acl(role -> allowed_endpoints)` 进行授权
- `allowed_endpoints` 必须以 `^/kbs` 开头并以 `$` 结尾

示例：

```toml
[admin]
authorization_mode = "AuthenticatedAuthorization"

[admin.authentication.bearer_jwt]
identity_providers = [
  { issuer = "authentik", audience = "kbs-admin", jwk_set_uri = "https://auth.example.com/application/o/kbs/jwks/" },
  { issuer = "legacy-admin", public_key_uri = "file:///etc/kbs/legacy-admin-public.pem" },
  { issuer = "dev-admin", public_key_uri = "./test_data/admin/public_key.pem" }
]

[admin.authorization.regex_acl]
acls = [
  { role = "kbs-admin", allowed_endpoints = "^/kbs/v0/resource/.+$" },
  { role = "kbs-admin", allowed_endpoints = "^/kbs/v0/resource-policy$" },
  { role = "kbs-admin", allowed_endpoints = "^/kbs/v0/attestation-policy$" }
]
```

#### 对外部身份 / 授权生态的支持

`AuthenticatedAuthorization` 模式可以与外部身份生态协同工作，包括各类开源平台：

- 由外部 IdP 签发 JWT
- KBS 使用配置好的 JWKS 和/或 PEM 公钥校验 JWT 签名
- 若配置了 `identity_providers`，KBS 会检查 JWT 的 `iss` 与 `aud`
- JWT 必须包含 `role` claim，KBS 会将其映射到本地管理 ACL 角色
- KBS 通过正则 ACL 规则对每个管理 API 路径进行授权

这样可以让身份签发保持在外部，同时使 KBS 侧的授权保持显式且可审计。

## `bearer_jwt` 配置参考

`[admin.authentication.bearer_jwt]` 接受以下配置：

- `identity_providers`（数组）：受信任的身份提供方列表

每个 `identity_providers` 条目：

- `issuer`（可选字符串）：期望的 JWT `iss` 值；为空表示不做 issuer 校验
- `audience`（可选字符串）：期望的 JWT `aud` 值；为空表示不做 audience 校验
- `public_key_uri`（可选字符串）：PEM 公钥来源
- `jwk_set_uri`（可选字符串）：受信任的 JWKS 来源（格式见下文）

每个条目必须至少提供 `public_key_uri` 或 `jwk_set_uri` 其中之一。

`jwk_set_uri` 的格式：

- `file://...` 或本地路径（例如 `./keys/admin.jwks`）：JWKS JSON 文件，直接读取
- `https://...`：远程 JWKS URL 或 OpenID issuer 基础 URL（见下方说明）

> [!NOTE]
> 对于远程 `jwk_set_uri`，KBS 首先尝试直接从所配置的 URL 加载 JWKS
> （例如某个 `/jwks/` 端点）。如果失败，则回退到在
> `{uri}/.well-known/openid-configuration` 进行 OpenID discovery，并从返回的 `jwks_uri` 加载密钥。
