# KBS 配置文件

> 说明：本文件是 `kbs/docs/config.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

机密容器 KBS 的各项属性可以通过一份 TOML 格式的配置文件进行配置。

> 注意：也支持 YAML 和 JSON 等其他格式。`config` crate 支持的其他格式可能也可使用。本文档在配置示例中使用 TOML。

配置文件的位置通过 `-c` 或 `--config-file` 命令行选项，或者通过 `KBS_CONFIG_FILE` 环境变量传给 KBS 二进制程序。

## 可配置属性

下面各节列出了可以通过配置文件设置的 KBS 属性。

### HTTP 服务器配置

以下属性可在 `[http_server]` 小节下设置。

| 属性                   | 类型         | 说明                                      | 是否必填 | 默认值                  |
|------------------------|--------------|--------------------------------------------------|----------|--------------------------|
| `sockets`              | 字符串数组   | 监听的一个或多个套接字。                | 否       | `["127.0.0.1:8080"]`     |
| `insecure_http`        | 布尔         | KBS HTTP 端点不使用 TLS。                 | 否       | `false`                  |
| `private_key`          | 字符串       | 用于 HTTPS 的私钥文件路径。               | 否       | 无                       |
| `certificate`          | 字符串       | 用于 HTTPS 的证书文件路径。               | 否       | 无                       |
| `payload_request_size` | 整数         | 请求载荷大小，单位为兆字节。              | 否       | 2                        |
| `worker_count`         | 整数         | HTTP actix 工作线程数                     | 否       | 逻辑 CPU 核数 |
| `tls_profile`          | 字符串       | TLS 安全配置文件（见 [TLS 配置](#tls-configuration)） | 否 | `intermediate` |
| `tls_min_version`      | 字符串       | 最低 TLS 版本：`1.2` 或 `1.3`              | 否       | 取决于配置文件        |
| `tls_max_version`      | 字符串       | 最高 TLS 版本：`1.2` 或 `1.3`              | 否       | 取决于配置文件        |
| `tls_ciphers`          | 字符串       | TLS 密码套件（以冒号分隔的 OpenSSL 列表） | 否       | 取决于配置文件        |
| `tls_groups`           | 字符串       | TLS 密钥交换组（以冒号分隔的列表）        | 否       | 自动检测（含 PQC）   |

> 说明：上表中的锚点 `#tls-configuration` 指向本文档英文原小节标题，中文译文保留该锚点以维持链接可用。

#### TLS 配置

KBS 通过预定义的安全配置文件和自定义设置支持灵活的 TLS 配置。这些 TLS 配置文件基于 [Mozilla 的 TLS 建议](https://wiki.mozilla.org/Security/Server_Side_TLS)。

##### TLS 安全配置文件

通过 `tls_profile` 设置可使用四种安全配置文件：

| 配置文件       | TLS 版本  | 说明                                              | 适用场景                          |
|-----------------|--------------|----------------------------------------------------------|-----------------------------------|
| `old`           | 1.2, 1.3     | 与老客户端的遗留兼容性                  | 最大化客户端兼容性      |
| `intermediate`  | 1.2, 1.3     | 安全性与兼容性均衡（推荐）        | 大多数生产部署       |
| `modern`        | 仅 1.3       | 最高安全性，仅 TLS 1.3                           | 高安全环境        |
| `custom`        | 可配置       | 完全掌控 TLS 各项参数                         | 有特殊安全要求的场景 |

**推荐：** 除非你有特定的安全性或兼容性要求，否则大多数部署使用 `intermediate`（默认值）。

##### 自定义 TLS 配置

当使用 `tls_profile = "custom"` 时，你可以显式配置：

- **`tls_min_version`**：最低 TLS 协议版本（`"1.2"` 或 `"1.3"`）
- **`tls_max_version`**：最高 TLS 协议版本（`"1.2"` 或 `"1.3"`）
- **`tls_ciphers`**：以冒号分隔的 OpenSSL 密码套件列表
  - 仅 TLS 1.3（Modern 配置文件或 min=1.3）：使用 TLS 1.3 的密码套件名称
    - 示例：`TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256`
    - TLS 1.3 的密码套件只指定加密与哈希算法
  - 仅 TLS 1.2（max=1.2）：使用 TLS 1.2 的密码套件名称
    - 示例：`ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384`
    - TLS 1.2 的密码套件包含密钥交换、身份认证、加密与哈希
  - 同时支持 TLS 1.2 和 1.3（Old/Intermediate）：可以在同一个字符串里混用两种格式，以冒号分隔
- **`tls_groups`**：以冒号分隔的、用于密钥交换的受支持组列表（对 TLS 1.2 和 1.3 均适用）
  - 对于 TLS 1.2：配置用于 ECDHE 密码套件的椭圆曲线
  - 对于 TLS 1.3：配置全部密钥交换机制（因为 1.3 的密码套件不含密钥交换部分）
  - 常见的传统组：`X25519`、`X448`、`secp256r1`（P-256）、`secp384r1`（P-384）、`secp521r1`（P-521）
  - PQC 混合组（OpenSSL 3.5+）：`X25519MLKEM768`、`SecP256r1MLKEM768`、`X448MLKEM1024`、`SecP384r1MLKEM1024`
  - 示例：`X25519:secp256r1:secp384r1`（仅传统算法）
  - 示例：`X25519MLKEM768:X25519:secp256r1`（后量子 + 传统回退）
  - 完整列表参见 [OpenSSL groups 文档](https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set1_groups_list.html)

> [!NOTE]
> 自定义 TLS 字段可以与任何配置文件配合使用，用于覆盖特定设置。如果你在非 custom 配置文件下设置了自定义字段，会记录一条告警日志，但自定义设置优先生效。

##### 后量子密码（PQC）支持

当你的 OpenSSL 安装（3.5+）支持后量子密钥交换算法时，KBS 会自动检测并启用它们。如果没有显式配置 `tls_groups`，KBS 将：

1. 检测可用的最佳 PQC 算法（ML-KEM 混合组）
2. 若 PQC 不可用则回退到传统算法
3. 按以下优先级顺序使用：
   - `X25519MLKEM768`（优先）
   - `SecP256r1MLKEM768`（符合 FIPS）
   - `X448MLKEM1024`（高安全性）
   - `SecP384r1MLKEM1024`（FIPS + 高安全性）

若要显式配置 PQC，请使用 `custom` 配置文件配合 `tls_groups`：

```toml
[http_server]
tls_profile = "custom"
tls_min_version = "1.3"
tls_max_version = "1.3"
tls_groups = "X25519MLKEM768:X25519:secp256r1"
```

完整示例参见 [TLS 配置示例](#tls-configuration-examples) 一节。

### 证明令牌配置

证明令牌配置控制证明令牌的校验。当资源获取由 KBS 处理时这一点很重要。
通常请求会附带一个证明令牌，KBS 会首先校验该令牌。

关于校验何时发生、信任锚与 AS 令牌签名之间的关系以及部署示例的概述，请参见
[证明令牌校验](./attestation_token_verification.md)。

以下属性可在 `[attestation_token]` 小节下设置。

| 属性                   | 类型         | 说明                                                                                                                       | 默认值 |
|-----------------------|--------------|-----------------------------------------------------------------------------------------------------------------------------------|---------|
| `trusted_jwk_sets`    | 字符串数组   | 受信任的 JWKS 来源（`file://` 或 `https://`）。尽可能直接加载 JWKS，否则通过 OpenID discovery 加载。其中的密钥用于校验证明令牌。               | 空   |
| `trusted_certs_paths` | 字符串数组   | 用于验证证明令牌可信性的受信任证书文件（PEM 格式）                                            | 空   |
| `extra_teekey_paths`  | 字符串数组   | 用户自定义的、指向 JWT 载荷中 TEE 公钥的路径                                                                          | 空   |
| `insecure_header_jwk`        | 布尔      | 对 JWT 头部中的 JWK 跳过 `x5c`/`trusted_certs_paths` 背书校验；签名仍会被校验 | `false` |

每个 JWT 都包含一个 TEE 公钥。用户可以使用 `extra_teekey_paths` 字段额外指定该密钥在 JWT 中的路径。
`extra_teekey_paths` 的一个示例是 `/attester_runtime_data/tee-pubkey`，它指向 JWT 载荷 claim 中的
`attester_runtime_data.tee-pubkey` 这个键。默认情况下，CoCo AS 令牌与 Intel TA
令牌的 TEE 公钥路径均已被支持。

对于 CoCo-AS 这类证明服务，JWT 头部通常携带一个 `jwk`（有时附带
`x5c` 证书链）。KBS 使用该密钥来校验令牌签名。是否检查该密钥的来源取决于 `insecure_header_jwk`：

- 若 `insecure_header_jwk` 为 `true`，则头部 `jwk` 被直接使用；KBS 不会依据 `trusted_certs_paths`
  校验其 `x5c` 链。JWT 签名仍会被校验。仅用于测试。
- 若 `insecure_header_jwk` 为 `false`，则头部 `jwk` 必须包含一个非空的 `x5c` 链，
  该链要与密钥相匹配，并能链接到 `trusted_certs_paths` 中的某个证书。
  若 `trusted_certs_paths` 为空，则带有头部 `jwk` 的令牌无法被校验。

对于 Intel TA 这类证明服务，JWT 头部携带的是 `kid` 而非内嵌的
`jwk`。该 `kid` 用于从 `trusted_jwk_sets` 中查找签名密钥。
此查找路径不受 `insecure_header_jwk` 影响。

### 证明配置

证明配置定义 KBS 的 RCAR 协议将使用哪个证明服务。

以下属性可在 `[attestation_service]` 小节下设置。

具体的证明服务可通过 `type` 字段设置。支持的证明服务有：

- `coco_as_builtin`：内置在 KBS 二进制中的 CoCo AS
- `coco_as_grpc`：远程运行的 CoCo AS 服务
- `intel_ta`：Intel&reg; Trust Authority

由于 `type` 字段不同，可用属性也不同。

`timeout` 和 `policy_id_map` 适用于所有类型。后者是一张映射表，
把客户端可选的每个证明策略选择器（attestation-policy-selector）映射到一个或多个策略 ID：

```toml
[attestation_service.policy_id_map]
alice = ["alice-strict"]
bob = ["bob-cpu", "bob-gpu"]
```

#### 内置 CoCo AS

当 `type` 设为 `coco_as_builtin` 时，可设置以下属性。

> 内置 CoCo AS 仅在启用以下一个或多个特性时可用：
> `coco-as-builtin`、`coco-as-builtin-no-verifier`

| 属性                   | 类型                        | 说明                                              | 默认值                                                                                                       |
|----------------------------|-----------------------------|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| `timeout`                  | 整数                     | 证明会话的最长时间（单位：分钟） | 5                                                                                                             |
| `policy_id_map`            | 字符串数组的映射         | 客户端可选的 AS 策略，以证明策略选择器为键。参见 [RCAR `Request`][ps]| `{}`                                                          |
| `rvps_config`              | [RVPSConfiguration][2]      | RVPS 配置                                       | 参见 [RVPSConfiguration][2]                                                                                    |
| `attestation_token_broker` | [AttestationTokenBroker][1] | 证明结果令牌配置。                  | 参见 [AttestationTokenBroker][1]                                                                               |
| `verifier_config`          | 对象                      | 可选的验证器专属配置（例如 TPM）| 参见 [Verifier Configuration][vcfg]                                                                            |

[1]: #attestationtokenbroker
[2]: #rvps-configuration
[3]: #keyvaluestorage
[4]: #tokensignerconfig
[vcfg]: ../../attestation-service/docs/config.md#verifier-configuration
[ps]: ./kbs_attestation_protocol.md#request

##### AttestationTokenBroker（证明令牌代理）

| 属性         | 类型                   | 说明                                                                    | 是否必填 | 默认值                                                               |
|------------------|------------------------|--------------------------------------------------------------------------------|----------|-----------------------------------------------------------------------|
| `duration_min`   | 整数                | 证明结果令牌的有效期，单位为分钟。                           | 否       | `5`                                                                   |
| `issuer_name`    | 字符串                 | 证明结果令牌的签发者名称。                                   | 否       | `CoCo-Attestation-Service`                                            |
| `developer_name` | 字符串                 | 用作 EAR 中 Verifier ID 一部分的开发者名称            | 否       | `https://confidentialcontainers.org`                                  |
| `build_name`     | 字符串                 | 用作 EAR 中 Verifier ID 一部分的构建名称                | 否       | 由 Cargo 包与 AS 版本自动生成             |
| `profile_name`   | 字符串                 | 描述该 EAR 令牌的 Profile                                       | 否       | `tag:github.com,2024:confidential-containers/Trustee`                  |
| `signer`         | [TokenSignerConfig][4] | 证明结果令牌的签名材料。                              | 否       | 无                                                                  |
| `verbose_token`  | 布尔                | 在 EAR 中包含详细信息，例如原始的附加设备证据。设备证据（例如 GPU）可能使令牌超过 actix-http 的 128 KiB 请求头限制（导致 CDH 的 `Authorization: Bearer` 资源 GET 出现 `message head is too large` 之类的错误）。设为 `false` 可从令牌中省略这些额外细节；校验仍会执行。 | 否       | `true`                                                                |

##### TokenSignerConfig（令牌签名配置）

该小节是**可选的**。省略时，会生成并使用一个临时 RSA 密钥对。

| 属性    | 类型   | 说明                                              | 是否必填 |
|-------------|--------|----------------------------------------------------------|----------|
| `key_path`  | 字符串 | RSA 密钥对文件（PEM 格式）路径。                     | 是      |
| `cert_url`  | 字符串 | RSA 公钥证书链（PEM 格式）的 URL。       | 否       |
| `cert_path` | 字符串 | RSA 公钥证书链（PEM 格式）的文件路径。 | 否       |

##### RVPS 配置

| 属性 | 类型   | 说明                                                                                | 是否必填 | 默认值   |
|----------|--------|--------------------------------------------------------------------------------------------|----------|-----------|
| `type`   | 字符串 | 可以是 `BuiltIn`（内置 RVPS）或 `GrpcRemote`（连接到远程 gRPC RVPS） | 否       | `BuiltIn` |

##### 内置 RVPS

如果 `type` 设为 `BuiltIn`，可以设置以下额外属性：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `extractors` | 对象 | 来源提取器（provenance extractors）的可选配置 | 否 | 无 |
| `storage_type` | 字符串 | 可选的 RVPS 专属存储类型覆盖。若提供，则仅针对 RVPS 覆盖 `storage_backend.storage_type`。后端专属参数仍复用 `storage_backend.backends` 中的配置。 | 否 | 无 |

> [!NOTE]
> **存储配置：** 默认情况下，内置 RVPS 使用统一的 `storage_backend` 配置（参见 [存储后端配置](#storage-backend-configuration)），并使用 `reference_value` 命名空间。不过，你也可以选择性地提供 `storage_type` 字段，仅覆盖 RVPS 使用的存储类型。
> 这允许你为不同组件使用不同的存储后端（例如 KBS 资源用 LocalFs，而 RVPS 参考值用 LocalJson）。

关于提取器配置的详细信息，包括可用的提取器及其选项，请参见 [RVPS README](../../rvps/README.md#extractors-configuration)。

##### 远程 RVPS

如果 `type` 设为 `GrpcRemote`，可以设置以下额外属性：

| 属性  | 类型   | 说明                       | 是否必填 | 默认值           |
|-----------|--------|-----------------------------------|----------|-------------------|
| `address` | 字符串 | RVPS 服务器的远程地址 | 否       | `127.0.0.1:50003` |

#### gRPC CoCo AS（gRPC 形态）

当 `type` 设为 `coco_as_grpc` 时，KBS 会尝试连接远程的 CoCo AS 来完成证明。
可以设置以下属性。

> gRPC CoCo AS 仅在启用 `coco-as-grpc` 特性时可用。

| 属性    | 类型    | 说明                                                                                                                   | 默认值                  |
|-------------|---------|-------------------------------------------------------------------------------------------------------------------------------|--------------------------|
| `timeout`   | 整数 | RCAR 握手的 `auth` 与 `attest` 请求之间的最长时间（单位：分钟）                                           | 5                        |
| `policy_id_map` | 字符串数组的映射 | 客户端可选的 AS 策略，以证明策略选择器为键。参见 [RCAR `Request`][ps]                    | `{}`                     |
| `as_addr`   | 字符串  | 远程 CoCoAS 的 URL                                                                                                  | `http://127.0.0.1:50004` |
| `pool_size` | 整数 | KBS 与 CoCoAS 之间的连接通过连接池维护。该属性决定连接池的最大容量 | `100`                    |

#### Intel&reg; TA（英特尔信任权威）

当 `type` 设为 `intel_ta` 时，KBS 会尝试连接远程的 Intel TA 服务来完成证明。
可以设置以下属性。

> Intel Trust Authority AS 仅在启用 `intel-trust-authority-as` 特性时可用。

| 属性                 | 类型         | 说明                                                                                            | 是否必填 | 默认值 |
|--------------------------|--------------|--------------------------------------------------------------------------------------------------------|----------|---------|
| `timeout`                | 整数      | RCAR 握手的 `auth` 与 `attest` 请求之间的最长时间（单位：分钟）                    | 否       | 5       |
| `base_url`               | 字符串       | Intel Trust Authority API URL。                                                                         | 是      | -       |
| `api_key`                | 字符串       | Intel Trust Authority API key。                                                                         | 是      | -       |
| `certs_file`             | 字符串       | Intel Trust Authority 门户的 URL，或用于令牌校验的 JWKS 文件路径。               | 是      | -       |
| `policy_ids`             | 字符串数组 | 在 ITA 门户中定义的策略 ID 列表，需加引号并以逗号分隔。                                   | 否       | `[]`    |
| `policy_id_map`          | 字符串数组的映射 | 客户端可选的 AS 策略，以证明策略选择器为键。某个被选中的证明策略选择器会替换 `policy_ids`。参见 [RCAR `Request`][ps] | 否       | `{}`    |
| `allow_unmatched_policy` | 布尔      | 是否要求策略匹配。若未指定 `policy_ids`，则不检查策略匹配。 | 否       | false   |

详细[文档](https://docs.trustauthority.intel.com)。

### 管理 API 配置

管理模式在 `[admin]` 下配置，可选值为：

- `authorization_mode = "InsecureAllowAll"`
- `authorization_mode = "DenyAll"`
- `authorization_mode = "AuthenticatedAuthorization"`（生产环境推荐）

对于 `authorization_mode = "AuthenticatedAuthorization"`，需配置：

- `[admin.authentication.bearer_jwt]`
- `[admin.authorization.regex_acl]`

`bearer_jwt` 属性：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `identity_providers` | 数组 | 用于 JWT 校验的受信任签发方条目 | 否 | 空 |

每个 `identity_providers` 条目：

| 属性 | 类型 | 说明 | 是否必填 |
|----------|------|-------------|----------|
| `issuer` | 字符串 | 期望的 JWT `iss` 值（留空则跳过 issuer 校验） | 否 |
| `audience` | 字符串 | 期望的 JWT `aud` 值（留空则跳过 audience 校验） | 否 |
| `public_key_uri` | 字符串 | PEM 公钥来源（`https://`、`file://`、本地路径） | 否* |
| `jwk_set_uri` | 字符串 | JWKS 来源（`https://`、`file://` 或本地路径）。远程 `https://` URL 尽可能直接加载 JWKS，否则通过 OpenID discovery 加载。 | 否* |

\* 至少需要 `public_key_uri` 或 `jwk_set_uri` 其中之一。

用于管理访问的 JWT **必须**包含 `role` claim。

`regex_acl` 属性：

| 属性 | 类型 | 说明 | 是否必填 |
|----------|------|-------------|----------|
| `acls` | 数组 | 角色到端点的放行规则 | 否 |

每条 ACL 条目：

| 属性 | 类型 | 说明 | 是否必填 |
|----------|------|-------------|----------|
| `role` | 字符串 | 要匹配的 JWT `role` 值 | 是 |
| `allowed_endpoints` | 字符串 | 允许的请求路径的正则表达式 | 是 |

`allowed_endpoints` 必须以 `^/kbs` 开头并以 `$` 结尾。

### 存储后端配置

KBS 支持统一的存储后端配置，允许你声明一份将被用于 KBS 全部存储需求的存储配置，包括
KBS 所依赖的所有持久化存储，其中还可能包括底层内置的 AS 与内置 RVPS。

这消除了为每个组件单独配置存储的需要，从而简化了部署。

关于统一存储后端配置格式的详细信息，包括**命名空间**是什么以及它如何工作，请参见
[Key-Value Storage README](../../deps/key-value-storage/README.md#unified-storage-backend-configuration)。

#### KBS 中的存储命名空间

使用统一存储后端配置时，KBS 会创建以下存储命名空间：

| 命名空间名称 | 组件 | 说明 |
|---------------|-----------|-------------|
| `kbs` | KBS 策略引擎 | 存储 KBS 所使用的东西，例如 KBS 资源策略 |
| `repository` | 资源插件 | 存储由资源插件管理的机密资源 |
| `kbs_protocol_session` | KBS 证明会话 | 存储 RCAR 证明会话状态 |
| `attestation_service_policy` | 内置 AS | 存储内置证明服务的 EAR 策略 |
| `reference_value` | 内置 AS RVPS | 存储内置 RVPS 的参考值 |

KBS 资源策略会以键 `resource-policy` 存储在 `kbs` 命名空间内。

详细的配置选项与示例请参见 [Key-Value Storage README](../../deps/key-value-storage/README.md#unified-storage-backend-configuration)。

> [!NOTE]
> 所有持久化存储都通过 `[storage_backend]` 配置。若省略 `[storage_backend]`，则使用默认的内存存储（数据在重启后不会保留）。

#### 可选的会话存储类型覆盖

KBS 为证明会话状态支持一个可选的 `session_storage_type` 字段。

若未配置 `session_storage_type`，KBS 会回退到 `storage_backend.storage_type`。

后端专属配置始终复用 `storage_backend.backends` 中的配置。

证明会话存储的命名空间始终为 `kbs_protocol_session`。

### 插件配置

KBS 支持不同类型的插件，可以通过添加相应的配置来启用它们。

同时允许存在多个 `[[plugins]]` 小节，用于配置不同的插件。
具体的插件可通过 `name` 字段指定。

#### 资源配置

`name` 字段为 `resource` 时启用该插件。

资源插件允许持有合法证明令牌的用户访问 KBS 所保管的存储。
在旧版本中也称之为 "Repository"。需要配置的属性如下。

| 属性 | 类型   | 说明                                                   | 是否必填 | 默认值    |
|----------|--------|---------------------------------------------------------------|----------|------------|
| `type`| 字符串 | 资源的存储类型：`kvstorage`、`Aliyun`、`Aws`、`Gcp`、`Vault` | 否       | `kvstorage`|

当 `storage_backend_type = "kvstorage"`（默认）时，资源插件使用统一的[存储后端](#storage-backend-configuration)，命名空间为 `repository`。只需在 `[storage_backend]` 小节中配置存储。

当 `storage_backend_type = "Aliyun"` 时：

| 属性          | 类型   | 说明                       | 是否必填 | 示例                                             |
|-------------------|--------|-----------------------------------|----------|-----------------------------------------------------|
| `client_key`      | 字符串 | KMS 实例的 AAP client key | 是      | `{"KeyId": "KA..", "PrivateKeyData": "MIIJqwI..."}` |
| `kms_instance_id` | 字符串 | KMS 实例 ID               | 是      | `kst-shh668f7...`                                   |
| `password`        | 字符串 | AAP client key 的密码           | 是      | `8f9989c18d27...`                                   |
| `cert_pem`        | 字符串 | KMS 实例的 CA 证书      | 是      | `-----BEGIN CERTIFICATE----- ...`                   |

当 `storage_backend_type = "Aws"` 时：

| 属性       | 类型   | 说明                                                                                                       | 是否必填 | 示例                                                |
|----------------|--------|-------------------------------------------------------------------------------------------------------------------|----------|--------------------------------------------------------|
| `region`       | 字符串 | AWS 区域。若省略，则从默认凭证链解析（`AWS_REGION`、profile、IMDS 等）。              | 否       | `us-east-1`                                            |
| `endpoint_url` | 字符串 | 端点覆盖。用于 FIPS 端点、GovCloud 或测试中的 LocalStack。                                      | 否       | `https://secretsmanager-fips.us-east-1.amazonaws.com`  |

凭证由 AWS 默认 provider 链解析（环境变量、共享
config、IRSA、EC2/ECS 实例角色）。该后端是只读的 —— 请通过 AWS API 来供给
机密。KBS 的 `tag` 字段映射到 Secrets Manager 的
`SecretId`（名称或 ARN）；`repo/type` 被忽略。

当 `storage_backend_type = "Gcp"` 时：

| 属性       | 类型   | 说明                                                                                          | 是否必填 | 示例              |
|----------------|--------|------------------------------------------------------------------------------------------------------|----------|----------------------|
| `project_id`   | 字符串 | 拥有这些机密的 GCP 项目 ID（或项目编号）。                                                     | 是      | `my-project`         |
| `endpoint_url` | 字符串 | 端点覆盖。用于私有服务端点或测试中的假服务器。                      | 否       | `http://localhost:8080` |

凭证通过 Application Default Credentials 解析
（`GOOGLE_APPLICATION_CREDENTIALS`、`gcloud auth application-default login`，或
GCE/GKE/Cloud Run 的元数据服务器）。该后端是只读的 —— 请通过 GCP API 供给机密。
KBS 的 `tag` 字段映射到 Secret Manager 的机密名称，并且返回最新已启用的版本
（`projects/<project_id>/secrets/<tag>/versions/latest`）；`repo/type` 被忽略。

当 `storage_backend_type = "Vault"` 时：

| 属性     | 类型          | 是否必填 | 说明                                 | 默认值    |
|--------------|---------------|----------|---------------------------------------------|------------|
| `vault_url`  | 字符串        | 是      | Vault 服务器 URL（HTTP 或 HTTPS）            | -          |
| `token`      | 字符串        | 是      | Vault 认证令牌                  | -          |
| `mount_path` | 字符串        | 否       | Vault KV v1 挂载路径                      | `"secret"` |
| `verify_ssl` | 布尔       | 否       | 启用/禁用 SSL 证书校验 | `false`    |
| `ca_certs`   | 字符串数组 | 否       | 自定义 CA 证书文件的路径        | `None`     |

#### Nebula CA 配置

可以通过在 KBS 配置中添加以下内容来启用 Nebula CA 插件。

```yaml
[ [ plugins ] ]
  name = "nebula-ca"
```

下面的属性可用于进一步配置该插件。它们都是可选的。

| 属性                   | 类型       | 说明                                                 | 默认值                                                  |
|----------------------------|------------|-------------------------------------------------------------|----------------------------------------------------------|
| `nebula_cert_bin_path`     | 字符串     | `nebula-cert` 二进制文件路径                                   | 若未提供，将在 $PATH 中查找 `nebula-cert` |
| `work_dir`                 | 字符串     | 该插件的工作目录，需要 `rw` 权限     | `/opt/confidential-containers/kbs/nebula-ca`             |
| `[plugins.self_signed_ca]` | 子小节 | 用于创建 Nebula CA 密钥与证书的属性 | 见下表                                          |

下面的属性可以在 `[plugins.self_signed_ca]` 下定义，用于覆盖其默认值。它们都是可选的。

| 属性            | 类型    | 说明                                                                                                                                                     | 默认值                    | 示例                                                   |
|---------------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------|-----------------------------------------------------------|
| `name`              | 字符串  | 证书颁发机构的名称                                                                                                                               | `Trustee Nebula CA plugin` |                                                           |
| `argon_iterations`  | 整数 | 用于加密私钥口令的 Argon2 迭代次数参数                                                                                           | 1                          |                                                           |
| `argon_memory`      | 整数 | 用于加密私钥口令的 Argon2 内存参数（单位 KiB）                                                                                      | 2097152                    |                                                           |
| `argon_parallelism` | 整数 | 用于加密私钥口令的 Argon2 并行度参数                                                                                          | 4                          |                                                           |
| `curve`             | 字符串  | EdDSA/ECDSA 曲线（25519, P256）                                                                                                                                 | `25519`                    |                                                           |
| `duration`          | 字符串  | 证书的有效时长。合法的时间单位为：<小时>"h"<分钟>"m"<秒>"s"                                                    | `8760h0m0s`                |                                                           |
| `groups`            | 字符串  | 以逗号分隔的组列表。这将限制下级证书可以使用哪些组                                                                          | ""                         | `server,ssh`                                              |
| `ips`               | 字符串  | 以逗号分隔的 ipv4 地址与 CIDR 记法网络列表。这将限制下级证书可以在 ip 地址中使用哪些 ipv4 地址与网络 | ""                         | `192.168.100.10/24,192.168.100.15/24`                     |
| `out_qr`            | 字符串  | 证书二维码图片（png）的写入路径                                                                                                          |                            | `/opt/confidential-containers/kbs/nebula-ca/ca/ca_qr.crt` |
| `subnets`           | 字符串  | 以逗号分隔的 ipv4 地址与 CIDR 记法网络列表。这将限制下级证书可以在子网中使用哪些 ipv4 地址与网络       | ""                         | `192.168.86.0/24`                                         |

Nebula CA 的密钥与证书存储在 `${work_dir}/ca/ca.{key,crt}` 中。如果这些文件是在
之前的运行中生成的，
或者是[带外生成的](https://nebula.defined.net/docs/guides/quick-start/#creating-your-first-certificate-authority)，
插件将直接（重新）使用它们；否则，插件会使用 `[plugins.self_signed_ca]` 中的属性调用 `nebula-cert` 二进制来生成新的密钥与证书。

详细[文档](#kbs/docs/plugins/nebula_ca.md)。

#### SPIRE 插件配置

可以通过在 KBS 配置中添加以下内容来启用 SPIRE 插件（需要 KBS 在构建时启用
`spire-plugin` cargo 特性）。

```toml
[[plugins]]
name = "spire"
admin_socket_path = "/run/spire/sockets/admin.sock"
```

| 属性                    | 类型   | 说明                                                                        | 默认值   |
|------------------------------|--------|--------------------------------------------------------------------------------------|-----------|
| `admin_socket_path`         | 字符串 | SPIRE Agent 管理（Delegated Identity API）Unix 套接字的文件系统路径      | -         |
| `attested_selector_value`   | 字符串 | 用于始终附加的 `trustee:<value>` 标记选择器的值                    | `true`    |
| `k8s_selector_prefix`       | 字符串 | 已知 k8s 工作负载证明选择器键所使用的选择器类型                     | `k8s`     |
| `trustee_selector_prefix`   | 字符串 | 非 k8s 选择器键所使用的选择器类型                                          | `trustee` |

详细[文档](#kbs/docs/plugins/spire.md)。

#### 外部插件配置

外部插件通过自定义的基于 gRPC 的端点来扩展 KBS。单个 `name = "external"` 的
`[[plugins]]` 条目通过 `backends` 内联数组管理所有后端：

```toml
[[plugins]]
name = "external"
backends = [
  { name = "my-plugin", endpoint = "https://localhost:50051", ca_cert_path = "/etc/kbs/plugin-ca.pem" },
]
```

每个后端可通过 `/kbs/v0/external/<name>/...` 访问。端点的 scheme
决定传输方式：`http://` 表示明文，`https://` 表示 TLS。未知键会被拒绝，
因此携带已被移除的 `tls_mode` 键的配置将无法解析。

**后端级别字段：**

| 字段 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `name` | string | 是 | — | 用于 URL 路由的子插件名 |
| `endpoint` | string | 是 | — | gRPC 端点（`http://` 表示不安全，`https://` 表示 TLS） |
| `ca_cert_path` | string | 视情况 | — | 用于校验插件服务端证书的 CA 证书。使用 `https://` 端点时必填，使用 `http://` 时则该字段会被拒绝 |
| `timeout_ms` | integer | 否 | — | 单个请求的超时时间，单位为毫秒 |

部署细节与 gRPC 协议请参见 [`ext_plugin.md`](ext_plugin.md)。

## 配置示例

### 使用存储后端

有了统一存储后端，你只需声明一份将被用于全部存储需求的存储配置：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
insecure_http = true

[admin]
authorization_mode = "InsecureAllowAll"

[attestation_token]

# Unified storage backend configuration
# This single configuration will be used for:
# - KBS policy engine (namespace: "kbs")
# - Resource plugin storage (namespace: "repository")
# - KBS attestation session storage (namespace: "kbs_protocol_session")
# - Built-in AS policy storage (namespace: "attestation_service_policy")
# - Built-in AS RVPS storage (namespace: "reference_value")
[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/storage"

[attestation_service]
type = "coco_as_builtin"

[attestation_service.attestation_token_broker]
duration_min = 5

[attestation_service.rvps_config]
type = "BuiltIn"
# Optional: configure extractors
# [attestation_service.rvps_config.extractors]
# swid_extractor = {}

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

### 使用 RVPS 专属存储配置

你可以让 RVPS 使用与 KBS 其余部分不同的存储后端：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
insecure_http = true

[admin]
type = "InsecureAllowAll"

[attestation_token]

# Unified storage backend for most components (LocalFs)
[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/storage"

[storage_backend.backends.local_json]
file_dir_path = "/var/lib/rvps/references"

[attestation_service]
type = "coco_as_builtin"

[attestation_service.attestation_token_broker]
duration_min = 5

[attestation_service.rvps_config]
type = "BuiltIn"

# RVPS-specific storage type override (LocalJson)
# Backend-specific parameters are still read from [storage_backend.backends]
storage_type = "LocalJson"

# Optional: configure extractors
[attestation_service.rvps_config.extractors]
swid_extractor = {}

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

### 使用远程 CoCo AS

```toml
[http_server]
insecure_http = true

[admin]
authorization_mode = "InsecureAllowAll"

[attestation_service]
type = "coco_as_grpc"
as_addr = "http://127.0.0.1:50004"

[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/storage"

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

使用 Intel Trust Authority 证明服务运行：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
private_key = "/etc/kbs-private.key"
certificate = "/etc/kbs-cert.pem"
insecure_http = false
payload_request_size = 2

[attestation_token]
trusted_jwk_sets = ["https://portal.trustauthority.intel.com"]

[attestation_service]
type = "intel_ta"
base_url = "https://api.trustauthority.intel.com"
api_key = "tBfd5kKX2x9ahbodKV1..."
certs_file = "https://portal.trustauthority.intel.com"
allow_unmatched_policy = true

[admin]
authorization_mode = "AuthenticatedAuthorization"

[admin.authentication.bearer_jwt]
identity_providers = [
  { public_key_uri = "/etc/kbs-admin.pub" }
]

[admin.authorization.regex_acl]
acls = [{ role = "admin", allowed_endpoints = "^/kbs/.+$" }]

[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/storage"

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

使用 Nebula CA 插件：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
insecure_http = true

[admin]
authorization_mode = "InsecureAllowAll"

[attestation_token]

[attestation_service]
type = "coco_as_builtin"

[attestation_service.attestation_token_broker]
duration_min = 5

[attestation_service.rvps_config]
type = "BuiltIn"

[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/storage"

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"

[[plugins]]
name = "nebula-ca"
# If the Nebula CA key and certificate don't exist yet, the plugin will create them
# using the default configurations, which can be overriden here,
# e.g. the duration of the root CA.
#[plugin.self_signed_ca]
#duration = "4380hm0s0"
```

以护照模式分发资源：

```toml
[http_server]
sockets = ["127.0.0.1:50002"]
insecure_http = true

[admin]
authorization_mode = "InsecureAllowAll"

[attestation_token]
trusted_certs_paths = ["./work/ca-cert.pem"]
insecure_header_jwk = false

[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "./work/storage"

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

### TLS 配置示例

#### 使用 Modern 配置文件（仅 TLS 1.3）

适用于只支持现代客户端的高安全环境：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
private_key = "/etc/kbs-private.key"
certificate = "/etc/kbs-cert.pem"
tls_profile = "modern"

[attestation_service]
type = "coco_as_grpc"
as_addr = "http://127.0.0.1:50001"

[admin]
type = "DenyAll"

[storage_backend]
storage_type = "memory"
```

#### 使用 Intermediate 配置文件（默认，推荐）

在安全性与支持 TLS 1.2 和 1.3 的兼容性之间取得平衡：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
private_key = "/etc/kbs-private.key"
certificate = "/etc/kbs-cert.pem"
tls_profile = "intermediate"  # Can be omitted (it's the default)

[attestation_service]
type = "coco_as_grpc"
as_addr = "http://127.0.0.1:50001"

[admin]
type = "DenyAll"

[storage_backend]
storage_type = "memory"
```

#### 使用自定义 TLS 配置

完全掌控 TLS 各项参数：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
private_key = "/etc/kbs-private.key"
certificate = "/etc/kbs-cert.pem"
tls_profile = "custom"
tls_min_version = "1.2"
tls_max_version = "1.3"
# TLS 1.3 cipher suites (only encryption+hash; key exchange configured via tls_groups)
tls_ciphers = "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256"
# Classical key exchange groups
tls_groups = "X25519:secp256r1:secp384r1"

[attestation_service]
type = "coco_as_grpc"
as_addr = "http://127.0.0.1:50001"

[admin]
type = "DenyAll"

[storage_backend]
storage_type = "memory"
```

#### 使用后量子密码（PQC）

配置带后量子密钥交换的 TLS 1.3：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
private_key = "/etc/kbs-private.key"
certificate = "/etc/kbs-cert.pem"
tls_profile = "custom"
tls_min_version = "1.3"
tls_max_version = "1.3"
# TLS 1.3 cipher suites (only encryption+hash; key exchange configured via tls_groups)
tls_ciphers = "TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256"
# Post-quantum key exchange groups with classical fallbacks
tls_groups = "X25519MLKEM768:X25519:secp256r1"

[attestation_service]
type = "coco_as_grpc"
as_addr = "http://127.0.0.1:50001"

[admin]
type = "DenyAll"

[storage_backend]
storage_type = "memory"
```

> [!NOTE]
> 后量子算法需要 OpenSSL 3.5 或更高版本。如果 PQC 算法不可用，KBS 会记录一条警告日志并回退到传统算法。
